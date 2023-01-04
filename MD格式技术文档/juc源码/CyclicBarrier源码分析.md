## 前言

本篇我们来看看另一个和它比较像的并发工具CyclicBarrier。

## 与CountdownLatch的区别

### 将count值递减的线程

在CountDownLatch中，执行`countDown`方法的线程和执行`await`方法的线程不是一类线程。例如，线程M，N需要等待线程A,B,C,D,E执行完成后才能继续往下执行，则线程A,B,C,D,E执行完成后都将调用`countDown`方法，使得最后count变为了0，最后一个将count值减为0的线程调用的`tryReleaseShared`方法会成功返回true，从而调用`doReleaseShared()`唤醒所有在`sync queue`中等待共享锁的线程，这里对应的就是M,N。所以，**在CountDownLatch中，执行`countDown`的线程不会被挂起，调用`await`方法的线程会阻塞等待共享锁。**

而在CyclicBarrier中，将count值递减的线程和执行await方法的线程是一类线程，它们在执行完递减count的操作后，如果count值不为0，则可能同时被挂起。例如，线程A,B,C,D,E需要互相等待，保证所有线程都执行完了之后才能一起通过。

**这就好像同一个班级出去春游，到一个景区后先自由活动，一段时间后在指定的地点集合，然后去下一个景点。这里这个指定集合的地点就是CyclicBarrier中的barrier，每一个人到达后都会执行await方法先将需要继续等待的人数\(count\)减1，然后\(在条件队列上\)挂起等待，当最后一个人到了之后，发现人已经到到齐了，则他负责执行barrierCommand\(例如向班主任汇报人已经到齐\)，接着就唤醒所有还在等待中的线程，开启新一代。**

### 是否能重复使用

CountDownLatch是一次性的，当count值被减为0后，不会被重置;  
而CyclicBarrier在线程通过栅栏后，会开启新的一代，count值会被重置。

### 锁的类别与所使用到的队列

CountDownLatch使用的是共享锁，count值不为0时，线程在`sync queue`中等待，自始至终只牵涉到`sync queue`，由于使用共享锁，唤醒操作不必等待锁释放后再进行，唤醒操作很迅速。  
CyclicBarrier使用的是独占锁，count值不为0时，线程进入`condition queue`中等待，当count值降为0后，将被`signalAll()`方法唤醒到`sync queue`中去，然后挨个去争锁（因为是独占锁），在前驱节点释放锁以后，才能继续唤醒后继节点。

## 核心属性

```
private static class Generation {
    boolean broken = false;
}

/** The lock for guarding barrier entry */
private final ReentrantLock lock = new ReentrantLock();
/** Condition to wait on until tripped */
private final Condition trip = lock.newCondition();
/** The number of parties */
private final int parties;
/* The command to run when tripped */
private final Runnable barrierCommand;
/** The current generation */
private Generation generation = new Generation();

/**
 * Number of parties still waiting. Counts down from parties to 0
 * on each generation.  It is reset to parties on each new
 * generation or when broken.
 */
private int count;
```

CyclicBarrier的核心属性共有6个，我们将它分为三组。

第一组：

```
private final int parties;
private int count;
```

注意，这两个属性都是用来表征线程的数量，`parties`代表了参与线程的总数，即需要一同通过barrier的线程数，它是final类型的，由构造函数初始化，在类被创建后就一直不变了；`count`属性和CountDownLatch中的count一样，代表还需要等待的线程数，初始值为`parties`，每当一个线程到来就减一，如果该值为0，则说明所有的线程都到齐了，大家可以一起通过barrier了。

第二组：

```
private final ReentrantLock lock = new ReentrantLock();
private final Condition trip = lock.newCondition();
private Generation generation = new Generation();
```

这一组代表了CyclicBarrier的基础实现，即CyclicBarrier是基于独占锁`ReentrantLock`和条件队列实现的，而不是共享锁，所有相互等待的线程都会在同样的条件队列`trip`上挂起，被唤醒后将会被添加到`sync queue`中去争取独占锁lock，获得锁的线程将继续往下执行。

这里还有一个Generation对象，从定义上可以看出，它只有一个boolean类型的`broken`属性，关于这个Generation，我们下面分析源码的时候再详细讲。

第三组：

```
private final Runnable barrierCommand;
```

这是一个Runnable对象，代表了一个任务。当所有线程都到齐后，在它们一同通过barrier之前，就会执行这个对象的run方法，因此，**它有点类似于一个钩子方法**。当然这个参数不是必须的，如果线程在通过barrier之前没有什么特别需要处理的事情，该值可以为null。

## 构造函数

CyclicBarrier有两个构造函数：

```
public CyclicBarrier(int parties) {
    this(parties, null);
}
```

```
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

其中，第一个构造函数本质上也是调用了第二个，即如果不传入Runnable对象，则`barrierCommand`的值默认为null。

我们可以看出，构造函数就是初始化了`parties`，`count`，`barrierCommand` 三个变量。

## 辅助方法

要理解CyclicBarrier，首先我们需要弄明白它的几个辅助方法。

首先需要理解的是“代”（Generation）的概念，由于CyclicBarrier是可重复使用的，我们把每一个新的barrier称为一“代”。这个怎么理解呢，打个比方：**一个过山车有10个座位，景区常常需要等够10个人了，才会去开动过山车。于是我们常常在栏杆（barrier）外面等，等凑够了10个人，工作人员就把栏杆打开，让10个人通过；然后再将栏杆归位，后面新来的人还是要在栏杆外等待。这里，前面已经通过的人就是一“代”，后面再继续等待的一波人就是另外一“代”，栏杆每打开关闭一次，就产生新一的“代”。**

在CyclicBarrier，开启新的一代使用的是nextGeneration方法：

### nextGeneration\(\)

```
private void nextGeneration() {
    // 唤醒当前这一代中所有等待在条件队列里的线程
    trip.signalAll();
    // 恢复count值，开启新的一代
    count = parties;
    generation = new Generation();
}
```

该方法用于开启新的“一代”，通常是被最后一个调用await方法的线程调用。在该方法中，我们的主要工作就是唤醒当前这一代中所有等待在条件队列里的线程，将count的值恢复为parties，以及开启新的一代。

### breakBarrier\(\)

breakBarrier即打破现有的栅栏，让所有线程通过：

```
private void breakBarrier() {
    // 标记broken状态
    generation.broken = true;
    // 恢复count值
    count = parties;
    // 唤醒当前这一代中所有等待在条件队列里的线程（因为栅栏已经打破了）
    trip.signalAll();
}
```

这个breakBarrier怎么理解呢，继续拿上面过上车的例子打比方，有时候某个时间段，景区的人比较少，等待过山车的人数凑不够10个人，眼看后面迟迟没有人再来，这个时候有的工作人员也会打开栅栏，让正在等待的人进来坐过山车。这里工作人员的行为就是`breakBarrier`，由于并不是在凑够10个人的情况下就开启了栅栏，我们就把这一代的`broken`状态标记为`true`。

### reset\(\)

reset方法用于将barrier恢复成初始的状态，它的内部就是简单地调用了breakBarrier方法和nextGeneration方法。

```
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

这里要注意的是，如果在我们执行该方法时有线程正等待在barrier上，则它将立即返回并抛出`BrokenBarrierException`异常。  
另外一点值得注意的是，该方法执行前需要先获得锁。

## await

看完前面的辅助方法之后，接下来我们就来看CyclicBarrier最核心的await方法，可以说整个CyclicBarrier最关键的只有它了。它也是一个**集“countDown”和“阻塞等待”于一体的方法。**

await方法有两种版本，一种带超时机制，一种不带，然而从源码上看，它们最终调用的都是带超时机制的dowait方法：

```
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

```
public int await(long timeout, TimeUnit unit) throws InterruptedException, BrokenBarrierException, TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
```

其中，dowait方法定义如下，它就是整个CyclicBarrier的核心了，我们直接在代码中以注释的形式分析：

```
private int dowait(boolean timed, long nanos) throws InterruptedException, BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    // 所有执行await方法的线程必须是已经持有了锁，所以这里必须先获取锁
    lock.lock();
    try {
        final Generation g = generation;

        // 前面说过，调用breakBarrier会将当前“代”的broken属性设为true
        // 如果一个正在await的线程发现barrier已经被break了，则将直接抛出BrokenBarrierException异常
        if (g.broken)
            throw new BrokenBarrierException();

        // 如果当前线程被中断了，则先将栅栏打破，再抛出InterruptedException
        // 这么做的原因是，所以等待在barrier的线程都是相互等待的，如果其中一个被中断了，那其他的就不用等了。
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        // 当前线程已经来到了栅栏前，先将等待的线程数减一
        int index = --count;
        
        // 如果等待的线程数为0了，说明所有的parties都到齐了
        // 则可以唤醒所有等待的线程，让大家一起通过栅栏，并重置栅栏
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    // 如果创建CyclicBarrier时传入了barrierCommand
                    // 说明通过栅栏前有一些额外的工作要做
                    command.run(); 
                ranAction = true;
                // 唤醒所有线程，开启新一代
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // 如果count数不为0，就将当前线程挂起，直到所有的线程到齐，或者超时，或者中断发生
        for (;;) {
            try {
                // 如果没有设定超时机制，则直接调用condition的await方法
                if (!timed)
                    trip.await();  // 当前线程在这里被挂起
                else if (nanos > 0L)
                    // 如果设了超时，则等待指定的时间
                    nanos = trip.awaitNanos(nanos); // 当前线程在这里被挂起，超时时间到了就会自动唤醒
            } catch (InterruptedException ie) {
                // 执行到这里说明线程被中断了
                // 如果线程被中断时还处于当前这一“代”，并且当前这一代还没有被broken,则先打破栅栏
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // 注意来到这里有两种情况
                    // 一种是g!=generation，说明新的一代已经产生了，所以我们没有必要处理这个中断，只要再自我中断一下就好，交给后续的人处理
                    // 一种是g.broken = true, 说明中断前栅栏已经被打破了，既然中断发生时栅栏已经被打破了，也没有必要再处理这个中断了
                    Thread.currentThread().interrupt();
                }
            }

            // 注意，执行到这里是对应于线程从await状态被唤醒了
            
            // 这里先检测broken状态，能使broken状态变为true的，只有breakBarrier()方法，到这里对应的场景是
            // 1. 其他执行await方法的线程在挂起前就被中断了
            // 2. 其他执行await方法的线程在还处于等待中时被中断了
            // 2. 最后一个到达的线程在执行barrierCommand的时候发生了错误
            // 4. reset()方法被调用
            if (g.broken)
                throw new BrokenBarrierException();

            // 如果线程被唤醒时，新一代已经被开启了，说明一切正常，直接返回
            if (g != generation)
                return index;

            // 如果是因为超时时间到了被唤醒，则打破栅栏，返回TimeoutException
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```

这个await方法虽然包揽了countDown、阻塞线程、唤醒线程、执行barrierCommand任务、开启新一代，处理中断等诸多任务，但是代码本身还是比较好懂的。

值得注意的是，**await方法是有返回值的，代表了线程到达的顺序**，第一个到达的线程的index为`parties \- 1`，最后一个到达的线程的index为`0`

## 工具方法

除了重头戏await方法和它的一些辅助方法，CyclicBarrier还为我们提供了一些工具方法：

（1）获取参与的线程数parties

```
public int getParties() {
    return parties;
}
```

parties 在构造完成后就不会被修改了，因此对它的访问不需要加锁。

（2）获取正在等待中的线程数

```
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```

注意，这里加了锁，因为count方法可能会被多个线程同时修改。

（3）判断当前barrier是否已经broken

```
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```

注意，这里同样要加锁，因为broken属性可能被多个线程同时访问或修改。

## 实战

为了学以致用，接下来我们就来看看怎么使用这个并发工具，java官方文档为我们提供了一个使用的范例：

```
class Solver {
    final int N;
    final float[][] data;
    final CyclicBarrier barrier;

    class Worker implements Runnable {
        int myRow;

        Worker(int row) {
            myRow = row;
        }

        public void run() {
            while (!done()) {
                processRow(myRow);

                try {
                    barrier.await();
                } catch (InterruptedException ex) {
                    return;
                } catch (BrokenBarrierException ex) {
                    return;
                }
            }
        }
    }

    public Solver(float[][] matrix) {
        data = matrix;
        N = matrix.length;
        Runnable barrierAction =
                new Runnable() {
                    public void run() {
                        mergeRows(...);
                    }
                };
        barrier = new CyclicBarrier(N, barrierAction);

        List<Thread> threads = new ArrayList<Thread>(N);
        for (int i = 0; i < N; i++) {
            Thread thread = new Thread(new Worker(i));
            threads.add(thread);
            thread.start();
        }

        // wait until done
        for (Thread thread : threads)
            thread.join();
    }
}
```

在这个例子中，我们为传入的matrix数组的每一行都创建了一个线程进行处理，使用了CyclicBarrier来保证只有所有的线程都处理完之后，才会调用`mergeRows(...)`方法来合并结果。只要有一行没有处理完，所有的线程都会在`barrier.await()`处等待，最后一个执行完的线程将会负责唤醒所有等待的线程。

## 总结

- CyclicBarrier实现了类似CountDownLatch的逻辑，它可以使得一组线程之间相互等待，直到所有的线程都到齐了之后再继续往下执行。
- CyclicBarrier基于条件队列和独占锁来实现，而非共享锁。
- CyclicBarrier可重复使用，在所有线程都到齐了一起通过后，将会开启新的一代。
- CyclicBarrier使用了`“all-or-none breakage model”`，所有互相等待的线程，要么一起通过barrier，要么一个都不要通过，如果有一个线程因为中断，失败或者超时而过早的离开了barrier，则该barrier会被broken掉，所有等待在该barrier上的线程都会抛出`BrokenBarrierException`（或者`InterruptedException`）。
