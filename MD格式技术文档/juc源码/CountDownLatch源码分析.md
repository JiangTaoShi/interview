## 前言

CountDownLatch是一个很有用的工具，latch是门闩的意思，该工具是为了解决某些操作只能在一组操作全部执行完成后才能执行的情景。例如，小组早上开会，只有等所有人到齐了才能开；再如，游乐园里的过山车，一次可以坐10个人，为了节约成本，通常是等够10个人了才开。`CountDown`是倒数计数，所以CountDownLatch的用法通常是设定一个大于0的值，该值即代表需要等待的总任务数，每完成一个任务后，将总任务数减一，直到最后该值为0，说明所有等待的任务都执行完了，“门闩”此时就被打开，后面的任务可以继续执行。

CountDownLatch本身是基于共享锁实现的，如果你还不了解共享锁，建议先读一下共享锁的获取与释放，然后再继续往下看。

## 核心属性

CountDownLatch主要是通过AQS的共享锁机制实现的，因此它的核心属性只有一个sync，它继承自AQS，同时覆写了`tryAcquireShared`和`tryReleaseShared`，以完成具体的实现共享锁的获取与释放的逻辑。

```
private final Sync sync;
```

```
/**
 * Synchronization control For CountDownLatch.
 * Uses AQS state to represent count.
 */
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

## 构造函数

```
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

在构造函数中，我们就是简单传入了一个不小于0的任务数，由上面Sync的构造函数可知，这个任务数就是AQS的state的初始值。

## 核心方法

CountDownLatch最核心的方法只有两个，一个是countDown方法，每调用一次，就会将当前的count减一，当count值为0时，就会唤醒所有等待中的线程；另一个是await方法，它有两种形式，一种是阻塞式，一种是带超时机制的形式，该方法用于将当前等待“门闩”开启的线程挂起，直到count值为0，这一点很类似于条件队列，相当于等待的条件就是count值为0，**然而其底层的实现并不是用条件队列，而是共享锁。**

### countDown\(\)

```
public void countDown() {
    sync.releaseShared(1);
}
```

前面说过，countDown\(\)方法的目的就是将count值减一，并且在count值为0时，唤醒所有等待的线程，它内部调用的其实是释放共享锁的操作：

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

该方法由AQS实现，但是tryReleaseShared方法由Sync类自己实现：

```
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

该方法的实现很简单，就是获取当前的state值，如果已经为0了，直接返回false；否则通过CAS操作将state值减一，之后返回的是`nextc == 0`，由此可见，**该方法只有在count值原来不为0，但是调用后变为0时，才会返回true，否则返回false**，并且也可以看出，该方法在返回true之后，后面如果再次调用，还是会返回false。也就是说，**调用该方法只有一种情况会返回true，那就是state值从大于0变为0值时**，这时也是所有在门闩前的任务都完成了。

在`tryReleaseShared`返回true以后，将调用`doReleaseShared`方法唤醒所有等待中的线程，该方法我们在前面的文章中已经详细分析过了，这里就不再赘述了。

值得一提的是，我们其实并不关心`releaseShared`的返回值，而只关心`tryReleaseShared`的返回值，或者只关心count到0了没有，这里更像是借了共享锁的“壳”，来完成我们的目的，事实上我们完全可以自己设一个全局变量count来实现相同的效果，只不过对这个全局变量的操作也必须使用CAS。

### await\(\)

与Condition的await\(\)方法的语义相同，该方法是阻塞式地等待，并且是响应中断的，只不过它不是在等待signal操作，而是在等待count值为0：

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

可见，await方法内部调用的是acquireSharedInterruptibly方法，相当于借用了获取共享锁的“壳”：

```
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

我们来回忆一下独占模式下对应的方法：

```
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

可见，两者用的是同一个框架，只是这里：

- `tryAcquire(arg)` 换成了 `tryAcquireShared(arg)` \(子类实现\)
- `doAcquireInterruptibly(arg)` 换成了 `doAcquireSharedInterruptibly(arg)` （AQS提供）

我们先来看看Sync子类对于tryAcquireShared的实现：

```
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

**该方法似乎有点挂羊头卖狗肉的感觉——所谓的获取共享锁，事实上并不是什么抢锁的行为，没有任何CAS操作，它就是判断当前的state值是不是0，是就返回1，不是就返回-1。**

值得注意的是，在共享锁的获取与释放中我们特别提到过`tryAcquireShared`返回值的含义：

- 如果该值小于0，则代表当前线程获取共享锁失败
- 如果该值大于0，则代表当前线程获取共享锁成功，并且接下来其他线程尝试获取共享锁的行为很可能成功
- 如果该值等于0，则代表当前线程获取共享锁成功，但是接下来其他线程尝试获取共享锁的行为会失败

所以，当该方法的返回值不小于0时，就说明抢锁成功，可以直接退出了，所对应的就是count值已经为0，所有等待的事件都满足了。否则，我们调用`doAcquireSharedInterruptibly(arg)`将当前线程封装成Node，丢到`sync queue`中去阻塞等待：

```
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在前面我们介绍共享锁的获取时，已经分析过了`doAcquireShared`方法，只是它是不抛出`InterruptedException`的，`doAcquireSharedInterruptibly(arg)`是它的可中断版本，我们可以直接对比一下：  
![doAcquireShared-vs-doAcquireSharedInterruptibly](https://image-static.segmentfault.com/262/565/2625656068-5ba9b85e021de_articlex "doAcquireShared-vs-doAcquireSharedInterruptibly")

可见，它们仅仅是在对待中断的处理方式上有所不同，其他部分都是一样的，由于`doAcquireShared`在前面的文章中我们已经详细分析过了，这里就不再赘述了。

### await\(long timeout, TimeUnit unit\)

相较于`await()`方法，`await(long timeout, TimeUnit unit)`提供了超时等待机制：

```
public boolean await(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

```
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 || doAcquireSharedNanos(arg, nanosTimeout);
}
```

```
private boolean doAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

注意，在`tryAcquireSharedNanos`方法中，我们用到了`doAcquireSharedNanos`的返回值，如果该方法因为超时而退出时，则将返回false。由于await\(\)方法是阻塞式的，也就是说没有获取到锁是不会退出的，因此它没有返回值，换句话说，如果它正常返回了，则一定是因为获取到了锁而返回； 而`await(long timeout, TimeUnit unit)`由于有了超时机制，它是有返回值的，返回值为true则表示获取锁成功，为false则表示获取锁失败。**`doAcquireSharedNanos`的这个返回值有助于我们理解该方法究竟是因为获取到了锁而返回，还是因为超时时间到了而返回。**

至于`doAcquireSharedNanos`的实现细节，由于他和`doAcquireSharedInterruptibly`相比只是多了一个超时机制：  
![doAcquireSharedInterruptibly-vs-doAcquireSharedNanos](https://image-static.segmentfault.com/275/181/2751817432-5ba9b85de3b30_articlex "doAcquireSharedInterruptibly-vs-doAcquireSharedNanos")

代码本身很简单，就不赘述了。

## 实战

接下来我们来学习一个使用`CountDownLatch`的实际例子，Java的官方源码已经为我们提供了一个使用的示例代码：

```
class Driver { // ...
    void main() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);

        for (int i = 0; i < N; ++i) // create and start threads
            new Thread(new Worker(startSignal, doneSignal)).start();

        doSomethingElse();            // don't let run yet
        startSignal.countDown();      // let all threads proceed
        doSomethingElse();
        doneSignal.await();           // wait for all to finish
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }

    public void run() {
        try {
            startSignal.await();
            doWork();
            doneSignal.countDown();
        } catch (InterruptedException ex) {
        } // return;
    }

    void doWork() { ...}
}
```

在这个例子中，有两个“闸门”，一个是`CountDownLatch startSignal = new CountDownLatch(1)`，它开启后，等待在这个“闸门”上的任务才能开始运行；另一个“闸门”是`CountDownLatch doneSignal = new CountDownLatch(N)`, 它表示等待N个任务都执行完成后，才能继续往下。

Worker实现了Runnable接口，代表了要执行的任务，在它的run方法中，我们先调用了`startSignal.await()`，等待`startSignal`这一“闸门”开启，闸门开启后，我们就执行自己的任务，任务完成后再执行`doneSignal.countDown()`，将等待的总任务数减一。

代码本身的逻辑非常简单好懂，这里不赘述了。

## 总结

- CountDownLatch相当于一个“门栓”，一个“闸门”，只有它开启了，代码才能继续往下执行。通常情况下，如果当前线程需要等其他线程执行完成后才能执行，我们就可以使用CountDownLatch。
- 使用`CountDownLatch#await`方法阻塞等待一个“闸门”的开启。
- 使用`CountDownLatch#countDown`方法减少闸门所等待的任务数。
- CountDownLatch基于**共享锁**实现。
- CountDownLatch是一次性的，“闸门”开启后，无法再重复使用，如果想重复使用，应该用\[CyclicBarrier\]\(\)
