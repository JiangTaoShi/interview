## 前言

上一篇我们学习了lock接口，本篇我们就以ReentrantLock为例，学习一下Lock锁的基本的实现。我们先来看看Lock接口中的方法与ReentrantLock对其实现的对照表：

| Lock 接口 | ReentrantLock 实现 |
| --- | --- |
| lock\(\) | sync.lock\(\) |
| lockInterruptibly\(\) | sync.acquireInterruptibly\(1\) |
| tryLock\(\) | sync.nonfairTryAcquire\(1\) |
| tryLock\(long time, TimeUnit unit\) | sync.tryAcquireNanos\(1, unit.toNanos\(timeout\)\) |
| unlock\(\) | sync.release\(1\) |
| newCondition\(\) | sync.newCondition\(\) |

从表中可以看出，ReentrantLock对于Lock接口的实现都是直接“转交”给sync对象的。

## 核心属性

ReentrantLock只有一个sync属性，别看只有一个属性，这个属性提供了所有的实现，我们上面介绍ReentrantLock对Lock接口的实现的时候就说到，它对所有的Lock方法的实现都调用了sync的方法，这个sync就是ReentrantLock的属性，它继承了AQS.

```
private final Sync sync;
```

```
abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();
    //...
}
```

在Sync类中，定义了一个抽象方法lock，该方法应当由继承它的子类来实现，关于继承它的子类，我们在下一节分析构造函数时再看。

## 构造函数

ReentrantLock共有两个构造函数：

```
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认的构造函数使用了非公平锁，另外一个构造函数通过传入一个boolean类型的`fair`变量来决定使用公平锁还是非公平锁。其中，FairSync和NonfairSync的定义如下：

```
static final class FairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}

static final class NonfairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}
```

这里为什么默认创建的是非公平锁呢？因为非公平锁的效率高呀，当一个线程请求非公平锁时，如果在**发出请求的同时**该锁变成可用状态，那么这个线程会跳过队列中所有的等待线程而获得锁。有的同学会说了，这不就是插队吗？  
没错，这就是插队！这也就是为什么它被称作非公平锁。  
之所以使用这种方式是因为：

> 在恢复一个被挂起的线程与该线程真正运行之间存在着严重的延迟。

在公平锁模式下，大家讲究先来后到，如果当前线程A在请求锁，即使现在锁处于可用状态，它也得在队列的末尾排着，这时我们需要唤醒排在等待队列队首的线程H\(在AQS中其实是次头节点\)，由于恢复一个被挂起的线程并且让它真正运行起来需要较长时间，那么这段时间锁就处于空闲状态，时间和资源就白白浪费了，非公平锁的设计思想就是将这段白白浪费的时间利用起来——由于线程A在请求锁的时候本身就处于运行状态，因此如果我们此时把锁给它，它就会立即执行自己的任务，因此线程A有机会在线程H完全唤醒之前获得、使用以及释放锁。这样我们就可以把线程H恢复运行的这段时间给利用起来了，结果就是线程A更早的获取了锁，线程H获取锁的时刻也没有推迟。因此提高了吞吐量。

当然，非公平锁仅仅是在**当前线程请求锁，并且锁处于可用状态时**有效，当请求锁时，锁已经被其他线程占有时，就只能还是老老实实的去排队了。

无论是非公平锁的实现NonfairSync还是公平锁的实现FairSync，它们都覆写了lock方法和tryAcquire方法，这两个方法都将用于获取一个锁。

## Lock接口方法实现

### lock\(\)

#### 公平锁实现

关于ReentrantLock对于lock方法的公平锁的实现逻辑，我们在独占锁的获取中已经讲过了，这里不再赘述。如果你还没有看过那篇文章或者还不了解AQS，建议先去看一下那一篇文章，然后再读下文。

#### 非公平锁实现

接下来我们看看非公平锁的实现逻辑：

```
// NonfairSync中的lock方法
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

对比公平锁中的lock方法：

```
// FairSync中的lock方法
final void lock() {
    acquire(1);
}
```

可见，相比公平锁，非公平锁在当前锁没有被占用时，可以直接尝试去获取锁，而不用排队，所以它在一开始就尝试使用CAS操作去抢锁，只有在该操作失败后，才会调用AQS的acquire方法。

由于acquire方法中除了tryAcquire由子类实现外，其余都由AQS实现，我们在前面的文章中已经介绍的很详细了，这里不再赘述，我们仅仅看一下非公平锁的tryAcquire方法实现：

```
// NonfairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

它调用了Sync类的nonfairTryAcquire方法：

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 只有这一处和公平锁的实现不同，其它的完全一样。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

我们可以拿它和公平锁的tryAcquire对比一下：

```
// FairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

看见没？这两个方法几乎一模一样，唯一的区别就是非公平锁在抢锁时不再需要调用`hasQueuedPredecessors`方法先去判断是否有线程排在自己前面，而是直接争锁，其它的完全和公平锁一致。

### lockInterruptibly\(\)

前面的lock方法是阻塞式的，抢到锁就返回，抢不到锁就将线程挂起，并且在抢锁的过程中是不响应中断的，lockInterruptibly提供了一种响应中断的方式，在ReentrantLock中，**无论是公平锁还是非公平锁，这个方法的实现都是一样的**：

```
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

他们都调用了AQS的`acquireInterruptibly`方法：

```
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

该方法首先检查当前线程是否已经被中断过了，如果已经被中断了，则立即抛出`InterruptedException`\(这一点是lockInterruptibly要求的)。

如果调用这个方法时，当前线程还没有被中断过，则接下来先尝试用普通的方法来获取锁（`tryAcquire`）。如果获取成功了，则万事大吉，直接就返回了；否则，与前面的lock方法一样，我们需要将当前线程包装成Node扔进等待队列，所不同的是，这次，在队列中尝试获取锁时，如果发生了中断，我们需要对它做出响应, 并抛出异常

```
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return; //与acquireQueued方法的不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException(); //与acquireQueued方法的不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

如果你在上面分析lock方法的时候已经理解了acquireQueued方法，那么再看这个方法就很轻松了，我们把lock方法中的`acquireQueued`拿出来和上面对比一下：

```
acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
```

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; //不同之处
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; //不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; //不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

通过代码对比可以看出，`doAcquireInterruptibly`和`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`的调用本质上讲并无区别。只不过对于`addWaiter(Node.EXCLUSIVE)`，一个是外部调用，通过参数传进来；一个是直接在方法内部调用。所以这两个方法的逻辑几乎是一样的，唯一的不同就是在`doAcquireInterruptibly`中，当我们检测到中断后，不再是简单的记录中断状态，而是直接抛出`InterruptedException`。

当抛出中断异常后，在返回前，我们将进入finally代码块进行善后工作，很明显，此时failed是为true的，我们将调用`cancelAcquire`方法：

```
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // 由当前节点向前遍历，跳过那些已经被cancel的节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    
    // 从当前节点向前开始查找，找到第一个waitStatus>0的Node, 该节点为pred
    // predNext即是pred节点的下一个节点
    // 到这里可知，pred节点是没有被cancel的节点，但是pred节点往后，一直到当前节点Node都处于被Cancel的状态
    Node predNext = pred.next;

    //将当前节点的waitStatus的状态设为Node.CANCELLED
    node.waitStatus = Node.CANCELLED;

    // 如果当前节点是尾节点，则将之前找到的节点pred重新设置成尾节点，并将pred节点的next属性由predNext修改成Null
    // 这一段本质上是将pred节点后面的节点全部移出队列，因为它们都被cancel掉了
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // 到这里说明当前节点已经不是尾节点了，或者设置新的尾节点失败了
        // 我们前面说过，并发条件下，什么都有可能发生
        // 即在当前线程运行这段代码的过程中，其他线程可能已经入队了，成为了新的尾节点
        // 虽然我们之前已经将当前节点的waitStatus设为了CANCELLED 
        // 但是由我们在分析lock方法的文章可知，新的节点入队后会设置闹钟，将找一个没有CANCEL的前驱节点，将它的status设置成SIGNAL以唤醒自己。
        // 所以，在当前节点的后继节点入队后，可能将当前节点的waitStatus修改成了SIGNAL
        // 而在这时，我们发起了中断，又将这个waitStatus修改成CANCELLED
        // 所以在当前节点出队前，要负责唤醒后继节点。
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

这个`cancelAcquire`方法不仅是取消了当前节点的排队，还会同时将当前节点之前的那些已经CANCEL掉的节点移出队列。不过这里尤其需要注意的是，这里是在并发条件下，此时此刻，新的节点可能已经入队了，成为了新的尾节点，这将会导致`node == tail && compareAndSetTail(node, pred)`这一条件失败。

这个函数的前半部分是就是基于当前节点就是队列的尾节点的，即在执行这个函数时，没有新的节点入队，这部分的逻辑比较简单，大家直接看代码中的注释解释即可。

而后半部分是基于有新的节点加进来，当前节点已经不再是尾节点的情况，我们详细看看这else部分：

```
if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
} else {
    int ws;
    if (pred != head &&
        ((ws = pred.waitStatus) == Node.SIGNAL ||
         (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
        pred.thread != null) {
        Node next = node.next;
        if (next != null && next.waitStatus <= 0)
            compareAndSetNext(pred, predNext, next); //将pred节点的后继节点改为当前节点的后继节点
    } else {
        unparkSuccessor(node);
    }

    node.next = node; // help GC
}
```

（这里再说明一下`pred`变量所代表的含义：它表示了从当前节点向前遍历所找到的第一个没有被cancel的节点。）

执行到else代码块，则我们目前的状况如下：

1.  当前线程被中断了，我们已经将它的Node的waitStatus属性设为CANCELLED，thread属性置为null
2.  在执行这个方法期间，又有其他线程加入到队列中来，成为了新的尾节点，使得当前线程已经不是队尾了

在这种情况下，我们将执行if语句，将pred节点的后继节点改为当前节点的后继节点\(`compareAndSetNext(pred, predNext, next)`\)，即将从pred节点开始（不包含pred节点）一直到当前节点（包括当前节点）之间的所有节点全部移出队列，因为他们都是被cancel的节点。当然这是基于一定条件的，条件为：

1.  pred节点不是头节点
2.  pred节点的thread不为null
3.  pred节点的waitStatus属性是SIGNAL或者是小于等于0但是被我们成功的设置成signal

上面这三个条件保证了pred节点确实是一个正在正常等待锁的线程，并且它的waitStatus属性为SIGNAL。  
如果这一条件无法被满足，那么我们将直接通过unparkSuccessor唤醒它的后继节点。

到这里，我们总结一下`cancelAcquire`方法：

1.  如果要cancel的节点已经是尾节点了，则在我们后面并没有节点需要唤醒，我们只需要从当前节点\(即尾节点\)开始向前遍历，找到所有已经cancel的节点，将他们移出队列即可
2.  如果要cancel的节点后面还有别的节点，并且我们找到的pred节点处于正常等待状态，我们还是直接将从当前节点开始，到pred节点直接的所有节点，全部移出队列，这里并不需要唤醒当前节点的后继节点，因为它已经接在了pred的后面，pred的waitStatus已经被置为SIGNAL，它会负责唤醒后继节点
3.  如果上面的条件不满足，按说明当前节点往前已经没有在等待中的线程了，我们就直接将后继节点唤醒。

有的同学就要问了，那第3条只是把当前节点的后继节点唤醒了，并没有将当前节点移除队列呀？但是当前节点已经取消排队了，不是应该移除队列吗？  
别着急，在后继节点被唤醒后，它会在抢锁时调用的`shouldParkAfterFailedAcquire`方法里面跳过已经CANCEL的节点，那个时候，当前节点就会被移出队列了。

### tryLock\(\)

由于tryLock仅仅是用于检查锁在当前调用的时候是不是可获得的，所以即使现在使用的是公平锁，在调用这个方法时，当前线程也会直接尝试去获取锁，哪怕这个时候队列中还有在等待中的线程。所以这一方法对于公平锁和非公平锁的实现是一样的，它被定义在Sync类中，由FairSync和NonfairSync直接继承使用：

```
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
```

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这个`nonfairTryAcquire`我们在上面分析非公平锁的lock方法时已经讲过了，这里只是简单的方法复用。**该方法不存在任何和队列相关的操作**，仅仅就是直接尝试去获锁，成功了就返回true，失败了就返回false。

可能大家会觉得公平锁也使用这种方式去tryLock就丧失了公平性，但是这种方式在某些情况下是非常有用的，如果你还是想维持公平性，那应该使用带超时机制的`tryLock`：

### tryLock\(long timeout, TimeUnit unit\)

与立即返回的`tryLock()`不同，`tryLock(long timeout, TimeUnit unit)`带了超时时间，所以是阻塞式的，并且在获取锁的过程中可以响应中断异常：

```
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

```
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
}
```

与`lockInterruptibly`方法一样，该方法首先检查当前线程是否已经被中断过了，如果已经被中断了，则立即抛出`InterruptedException`。

随后我们通过调用`tryAcquire`和`doAcquireNanos(arg, nanosTimeout)`方法来尝试获取锁，注意，这时公平锁和非公平锁对于`tryAcquire`方法就有不同的实现了，公平锁首先会检查当前有没有别的线程在队列中排队，关于公平锁和非公平锁对`tryAcquire`的不同实现上文已经讲过了，这里不再赘述。我们直接来看`doAcquireNanos`，这个方法其实和前面说的`doAcquireInterruptibly`方法很像，我们通过将相同的部分注释掉，直接看不同的部分：

```
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    /*final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;*/
                return true; // doAcquireInterruptibly中为 return
            /*}*/
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
       /* }
    } finally {
        if (failed)
            cancelAcquire(node);
    }*/
}
```

可以看出，这两个方法的逻辑大差不差，只是`doAcquireNanos`多了对于截止时间的检查。

不过这里有两点需要注意，一个是`doAcquireInterruptibly`是没有返回值的，而`doAcquireNanos`是有返回值的。这是因为`doAcquireNanos`有可能因为获取到锁而返回，也有可能因为超时时间到了而返回，为了区分这两种情况，因为超时时间而返回时，我们将返回false，代表并没有获取到锁。

另外一点值得注意的是，上面有一个`nanosTimeout > spinForTimeoutThreshold`的条件，在它满足的时候才会将当前线程挂起指定的时间，这个spinForTimeoutThreshold是个啥呢：

```
/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices
 * to improve responsiveness with very short timeouts.
 */
static final long spinForTimeoutThreshold = 1000L;
```

它就是个阈值，是为了提升性能用的。如果当前剩下的等待时间已经很短了，我们就直接使用自旋的形式等待，而不是将线程挂起，可见作者为了尽可能地优化AQS锁的性能费足了心思。

### unlock\(\)

unlock操作用于释放当前线程所占用的锁，这一点对于公平锁和非公平锁的实现是一样的，所以该方法被定义在Sync类中，由FairSync和NonfairSync直接继承使用：

```
public void unlock() {
    sync.release(1);
}
```

关于ReentrantLock的释放锁的操作，我们在独占锁的释放中已经详细的介绍过了，这里就不再赘述了。

### newCondition\(\)

ReentrantLock本身并没有实现Condition方法，它是直接调用了AQS的`newCondition`方法

```
public Condition newCondition() {
    return sync.newCondition();
}
```

而AQS的`newCondtion`方法就是简单地创建了一个`ConditionObject`对象：

```
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

关于`ConditionObject`对象的源码分析，请参见Condition接口实现

## 总结

ReentrantLock对于Lock接口方法的实现大多数是直接调用了AQS的方法，AQS中已经完成了大多数逻辑的实现，子类只需要直接继承使用即可，这足见AQS在并发编程中的地位。当然，有一些逻辑还是需要ReentrantLock自己去实现的，例如tryAcquire的逻辑。

AQS在并发编程中的地位举足轻重，只要弄懂了它，我们在学习其他并发编程工具的时候就会容易很多
