## 前言

本篇文章是基于线程间的同步与通信\(4\)——Lock 和 Condtion 这篇文章写的，在那篇文章中，我们分析了Condition接口所定义的方法，本篇我们就来看看AQS对于Condition接口的这些接口方法的具体实现。

## 概述

我们在前面介绍Conditon的时候说过，Condition接口的await/signal机制是设计用来代替监视器锁的wait/notify机制 的，因此，与监视器锁的wait/notify机制对照着学习有助于我们更好的理解Conditon接口:

| Object 方法 | Condition 方法 | 区别 |
| --- | --- | --- |
| void wait\(\) | void await\(\) |  |
| void wait\(long timeout\) | long awaitNanos\(long nanosTimeout\) | 时间单位，返回值 |
| void wait\(long timeout, int nanos\) | boolean await\(long time, TimeUnit unit\) | 时间单位，参数类型，返回值 |
| void notify\(\) | void signal\(\) |  |
| void notifyAll\(\) | void signalAll\(\) |  |
| \- | void awaitUninterruptibly\(\) | Condition独有 |
| \- | boolean awaitUntil\(Date deadline\) | Condition独有 |

这里先做一下说明，本文说wait方法时，是泛指`wait()`、`wait(long timeout)`、`wait(long timeout, int nanos)` 三个方法，当需要指明某个特定的方法时，会带上相应的参数。同样的，说notify方法时，也是泛指`notify(`\)，`notifyAll()`方法，await方法和signal方法以此类推。

首先，我们通过wait/notify机制来类比await/signal机制：

1.  调用wait方法的线程首先必须是已经进入了同步代码块，即已经获取了监视器锁；与之类似，调用await方法的线程首先必须获得lock锁
2.  调用wait方法的线程会释放已经获得的监视器锁，进入当前监视器锁的等待队列（`wait set`）中；与之类似，调用await方法的线程会释放已经获得的lock锁，进入到当前Condtion对应的条件队列中。
3.  调用监视器锁的notify方法会唤醒等待在该监视器锁上的线程，这些线程将开始参与锁竞争，并在获得锁后，从wait方法处恢复执行；与之类似，调用Condtion的signal方法会唤醒**对应的**条件队列中的线程，这些线程将开始参与锁竞争，并在获得锁后，从await方法处开始恢复执行。

## 实战

由于前面我们已经学习过了监视器锁的wait/notify机制，await/signal的用法基本类似。在正式分析源码之前，我们先来看一个使用condition的实例：

```java
class BoundedBuffer {
    final Lock lock = new ReentrantLock();
    final Condition notFull = lock.newCondition();
    final Condition notEmpty = lock.newCondition();

    final Object[] items = new Object[100];
    int putptr, takeptr, count;

    // 生产者方法，往数组里面写数据
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await(); //数组已满，没有空间时，挂起等待，直到数组“非满”（notFull）
            items[putptr] = x;
            if (++putptr == items.length) putptr = 0;
            ++count;
            // 因为放入了一个数据，数组肯定不是空的了
            // 此时唤醒等待这notEmpty条件上的线程
            notEmpty.signal(); 
        } finally {
            lock.unlock();
        }
    }

    // 消费者方法，从数组里面拿数据
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await(); // 数组是空的，没有数据可拿时，挂起等待，直到数组非空（notEmpty）
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            // 因为拿出了一个数据，数组肯定不是满的了
            // 此时唤醒等待这notFull条件上的线程
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

这是java官方文档提供的例子，是一个典型的生产者-消费者模型。这里在同一个lock锁上，创建了两个条件队列`notFull`, `notEmpty`。当数组已满，没有存储空间时，put方法在`notFull`条件上等待，直到数组“not full”;当数组空了，没有数据可读时，take方法在`notEmpty`条件上等待，直到数组“not empty”，而`notEmpty.signal()`和`notFull.signal()`则用来唤醒等待在这个条件上的线程。

注意，上面所说的，在`notFull` `notEmpty`条件上等待事实上是指线程在条件队列（condition queue）上等待，当该线程被相应的signal方法唤醒后，将进入到我们前面三篇介绍的`sync queue`中去争锁，争到锁后才能能await方法处返回。这里接牵涉到两种队列了——`condition queue`和`sync queue`，它们都定义在AQS中。

为了防止大家被AQS中的队列弄晕，这里我们先理理清：

## 同步队列 vs 条件队列

### sync queue

首先，在逐行分析AQS源码\(1\)——独占锁的获取这篇中我们说过，所有等待锁的线程都会被包装成Node扔到一个同步队列中。该同步队列如下：  
![dummy head](https://image-static.segmentfault.com/247/264/247264520-5ba118090d145_articlex "dummy head")

`sync queue`是一个双向链表，我们使用`prev`、`next`属性来串联节点。但是在这个同步队列中，我们一直没有用到`nextWaiter`属性，即使是在共享锁模式下，这一属性也只作为一个标记，指向了一个空节点，因此，在`sync queue`中，我们不会用它来串联节点。

### condtion queue

每创建一个Condtion对象就会对应一个Condtion队列，每一个调用了Condtion对象的await方法的线程都会被包装成Node扔进一个条件队列中，就像这样：  
![Conditon queue](https://image-static.segmentfault.com/215/704/2157044821-5ba2919246175_articlex "Conditon queue")  
可见，每一个Condition对象对应一个Conditon队列，每个Condtion队列都是独立的，互相不影响的。在上图中，如果我们对当前线程调用了`notFull.await()`, 则当前线程就会被包装成Node加到`notFull`队列的末尾。

值得注意的是，`condition queue`是一个单向链表，在该链表中我们使用`nextWaiter`属性来串联链表。但是，就像在`sync queue`中不会使用`nextWaiter`属性来串联链表一样，在`condition queue`中，也并不会用到`prev`, `next`属性，它们的值都为null。也就是说，在条件队列中，Node节点真正用到的属性只有三个：

- `thread`：代表当前正在等待某个条件的线程
- `waitStatus`：条件的等待状态
- `nextWaiter`：指向条件队列中的下一个节点

既然这里又提到了`waitStatus`，我们这里再回顾一下它的取值范围：

```
volatile int waitStatus;
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;
```

在条件队列中，我们只需要关注一个值即可——`CONDITION`。它表示线程处于正常的等待状态，而只要`waitStatus`不是`CONDITION`，我们就认为线程不再等待了，此时就要从条件队列中出队。

### sync queue 和 conditon queue的联系

一般情况下，等待锁的`sync queue`和条件队列`condition queue`是相互独立的，彼此之间并没有任何关系。但是，当我们调用某个条件队列的signal方法时，会将某个或所有等待在这个条件队列中的线程唤醒，被唤醒的线程和普通线程一样需要去争锁，如果没有抢到，则同样要被加到等待锁的`sync queue`中去，此时节点就从`condition queue`中被转移到`sync queue`中：  
![condtion queue to sync queue](https://image-static.segmentfault.com/310/844/3108441170-5ba2919246598_articlex "condtion queue to sync queue")

但是，这里尤其要注意的是，node是被**一个一个**转移过去的，哪怕我们调用的是`signalAll()`方法也是**一个一个**转移过去的，而不是将整个条件队列接在`sync queue`的末尾。

同时要注意的是，我们在`sync queue`中只使用`prev`、`next`来串联链表，而不使用`nextWaiter`;我们在`condition queue`中只使用`nextWaiter`来串联链表，而不使用`prev`、`next`.事实上，它们就是两个使用了同样的Node数据结构的完全独立的两种链表。因此，将节点从`condition queue`中转移到`sync queue`中时，我们需要断开原来的链接（`nextWaiter`）,建立新的链接（`prev`, `next`），这某种程度上也是需要将节点**一个一个**地转移过去的原因之一。

### 入队时和出队时的锁状态

`sync queue`是等待锁的队列，当一个线程被包装成Node加到该队列中时，必然是没有获取到锁；当处于该队列中的节点获取到了锁，它将从该队列中移除\(事实上移除操作是将获取到锁的节点设为新的dummy head,并将thread属性置为null\)。

condition队列是等待在特定条件下的队列，因为调用await方法时，必然是已经获得了lock锁，所以在进入condtion队列**前**线程必然是已经获取了锁；在被包装成Node扔进条件队列中后，线程将释放锁，然后挂起；当处于该队列中的线程被signal方法唤醒后，由于队列中的节点在之前挂起的时候已经释放了锁，所以必须先去再次的竞争锁，因此，该节点会被添加到`sync queue`中。因此，条件队列在出队时，线程并不持有锁。

所以事实上，这两个队列的锁状态正好相反：

- `condition queue`：入队时已经持有了锁 -> 在队列中释放锁 -> 离开队列时没有锁 -> 转移到sync queue
- `sync queue`：入队时没有锁 -> 在队列中争锁 -> 离开队列时获得了锁

通过上面的介绍，我们对条件队列已经有了感性的认识，接下来就让我们进入到本篇的重头戏——源码分析：

## CondtionObject

AQS对Condition这个接口的实现主要是通过ConditionObject，上面已经说个，它的核心实现就是是一个条件队列，每一个在某个condition上等待的线程都会被封装成Node对象扔进这个条件队列。

### 核心属性

它的核心属性只有两个：

```
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

这两个属性分别代表了条件队列的队头和队尾，每当我们新建一个conditionObject对象，都会对应一个条件队列。

### 构造函数

```
public ConditionObject() { }
```

构造函数啥也没干，可见，条件队列是延时初始化的，在真正用到的时候才会初始化。

## Condition接口方法实现

### await\(\)第一部分分析

```
public final void await() throws InterruptedException {
    // 如果当前线程在调动await()方法前已经被中断了，则直接抛出InterruptedException
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程封装成Node添加到条件队列
    Node node = addConditionWaiter();
    // 释放当前线程所占用的锁，保存当前的锁状态
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 如果当前队列不在同步队列中，说明刚刚被await, 还没有人调用signal方法，则直接将当前线程挂起
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 线程将在这里被挂起，停止运行
        // 能执行到这里说明要么是signal方法被调用了，要么是线程被中断了
        // 所以检查下线程被唤醒的原因，如果是因为中断被唤醒，则跳出while循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 第一部分就分析到这里，下面的部分我们到第二部分再看, 先把它注释起来
    /*
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    */
}
```

我们已经在上面的代码注释中描述了大体的流程，接下来我们详细来看看await方法中所调用方法的具体实现。

首先是将当前线程封装成Node扔进条件队列中的`addConditionWaiter`方法：

#### addConditionWaiter

```java
/**
 * Adds a new waiter to wait queue.
 * @return its new wait node
 */
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果尾节点被cancel了，则先遍历整个链表，清除所有被cancel的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    // 将当前线程包装成Node扔进条件队列
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    /*
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
    */
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

首先我们要思考的是，存在两个不同的线程同时入队的情况吗？不存在。为什么呢？因为前面说过了，能调用await方法的线程必然是已经获得了锁，而获得了锁的线程只有一个，所以这里不存在并发，因此不需要CAS操作。

在这个方法中，我们就是简单的将当前线程封装成Node加到条件队列的末尾。这和将一个线程封装成Node加入等待队列略有不同：

1.  节点加入`sync queue`时`waitStatus`的值为0，但节点加入`condition queue`时`waitStatus`的值为`Node.CONDTION`。
2.  `sync queue`的头节点为dummy节点，如果队列为空，则会先创建一个dummy节点，再创建一个代表当前节点的Node添加在dummy节点的后面；而`condtion queue` 没有dummy节点，初始化时，直接将`firstWaiter`和`lastWaiter`直接指向新建的节点就行了。
3.  `sync queue`是一个双向队列，在节点入队后，要同时修改**当前节点的前驱**和**前驱节点的后继**；而在`condtion queue`中，我们只修改了前驱节点的`nextWaiter`,也就是说，`condtion queue`是作为**单向队列**来使用的。

如果入队时发现尾节点已经取消等待了，那么我们就不应该接在它的后面，此时需要调用`unlinkCancelledWaiters`来剔除那些已经取消等待的线程：

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

该方法将从头节点开始遍历整个队列，剔除其中`waitStatus`不为Node.CONDTION的节点，这里使用了两个指针`firstWaiter`和`trail`来分别记录第一个和最后一个`waitStatus`不为Node.CONDTION的节点，这些都是基础的链表操作，很容易理解，这里不再赘述了。

#### fullyRelease

在节点被成功添加到队列的末尾后，我们将调用fullyRelease来释放当前线程所占用的锁：

```java
/**
 * Invokes release with current state value; returns saved state.
 * Cancels node and throws exception on failure.
 * @param node the condition node for this wait
 * @return previous sync state
 */
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

首先，当我们调用这个方法时，说明当前线程已经被封装成Node扔进条件队列了。在该方法中，我们通过release方法释放锁，还记得release方法吗，我们在独占锁的释放中已经详细讲过了，这里不再赘述了。

值得注意的是，这是一次性释放了所有的锁，即对于可重入锁而言，无论重入了几次，这里是一次性释放完的，这也就是为什么该方法的名字叫**fully**Release。但这里尤其要注意的是`release(savedState)`方法是有可能抛出IllegalMonitorStateException的，这是因为当前线程可能并不是持有锁的线程。但是咱前面不是说，只有持有锁的线程才能调用await方法吗？既然fullyRelease方法在await方法中，为啥当前线程还有可能并不是持有锁的线程呢？

虽然话是这么说，但是在调用await方法时，我们其实并没有检测`Thread.currentThread() == getExclusiveOwnerThread()`，换句话说，也就是执行到`fullyRelease`这一步，我们才会检测这一点，而这一点检测是由AQS子类实现tryRelease方法来保证的，例如，ReentrantLock对tryRelease方法的实现如下：

```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

当发现当前线程不是持有锁的线程时，我们就会进入finally块，将当前Node的状态设为Node.CANCELLED，这也就是为什么上面的`addConditionWaiter`在添加新节点前每次都会检查尾节点是否已经被取消了。

在当前线程的锁被完全释放了之后，我们就可以调用`LockSupport.park(this)`把当前线程挂起，等待被signal了。但是，在挂起当前线程之前我们先用`isOnSyncQueue`确保了它**不在**`sync queue`中，这是为什么呢？当前线程不是在一个和`sync queue`无关的条件队列中吗？怎么可能会出现在`sync queue`中的情况？

```java
/**
 * Returns true if a node, always one that was initially placed on
 * a condition queue, is now waiting to reacquire on sync queue.
 * @param node the node
 * @return true if is reacquiring
 */
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
     * node.prev can be non-null, but not yet on queue because
     * the CAS to place it on queue can fail. So we have to
     * traverse from tail to make sure it actually made it.  It
     * will always be near the tail in calls to this method, and
     * unless the CAS failed (which is unlikely), it will be
     * there, so we hardly ever traverse much.
     */
    return findNodeFromTail(node);
}
```

```
/**
 * Returns true if node is on sync queue by searching backwards from tail.
 * Called only when needed by isOnSyncQueue.
 * @return true if present
 */
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

为了解释这一问题，我们先来看看signal方法

### signalAll\(\)

在看signalAll之前，我们首先要区分调用signalAll方法的线程与signalAll方法要唤醒的线程（等待在对应的条件队列里的线程）：

- 调用signalAll方法的线程本身是已经持有了锁，现在准备释放锁了；
- 在条件队列里的线程是已经在对应的条件上挂起了，等待着被signal唤醒，然后去争锁。

首先，与调用notify时线程必须是已经持有了监视器锁类似，在调用condition的signal方法时，线程也必须是已经持有了lock锁：

```
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}
```

该方法首先检查当前调用signal方法的线程是不是持有锁的线程，这是通过`isHeldExclusively`方法来实现的，该方法由继承AQS的子类来实现，例如，ReentrantLock对该方法的实现为：

```
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

因为`exclusiveOwnerThread`保存了当前持有锁的线程，这里只要检测它是不是等于当前线程就行了。  
接下来先通过`firstWaiter`是否为空判断条件队列是否为空，如果条件队列不为空，则调用`doSignalAll`方法：

```
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

首先我们通过`lastWaiter = firstWaiter = null;`将整个条件队列清空，然后通过一个`do-while`循环，将原先的条件队列里面的节点**一个一个**拿出来\(令nextWaiter = null\)，再通过`transferForSignal`方法**一个一个**添加到`sync queue`的末尾：

```
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 */
final boolean transferForSignal(Node node) {
    // 如果该节点在调用signal方法前已经被取消了，则直接跳过这个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    // 如果该节点在条件队列中正常等待，则利用enq方法将该节点添加至sync queue队列的尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread); 
    return true;
}
```

在`transferForSignal`方法中，我们先使用CAS操作将当前节点的`waitStatus`状态由CONDTION设为0，如果修改不成功，则说明该节点已经被CANCEL了，则我们直接返回，操作下一个节点；如果修改成功，则说明我们已经将该节点从等待的条件队列中成功“唤醒”了，但此时该节点对应的线程并没有真正被唤醒，它还要和其他普通线程一样去争锁，因此它将被添加到`sync queue`的末尾等待获取锁。

我们这里通过`enq`方法将该节点添加进`sync queue`的末尾。关于该方法，我们在逐行分析AQS源码\(1\)——独占锁的获取中已经详细讲过了，这里不再赘述。不过这里尤其注意的是，enq方法将node节点添加进队列时，返回的是node的**前驱节点**。

在将节点成功添加进`sync queue`中后，我们得到了该节点在`sync queue`中的前驱节点。我们前面说过，在`sync queque`中的节点都要靠前驱节点去唤醒，所以，这里我们要做的就是将前驱节点的waitStatus设为Node.SIGNAL, 这一点和`shouldParkAfterFailedAcquire`所做的工作类似：

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

所不同的是，`shouldParkAfterFailedAcquire`将会向前查找，跳过那些被cancel的节点，然后将找到的第一个没有被cancel的节点的waitStatus设成SIGNAL，最后再挂起。而在`transferForSignal`中，当前Node所代表的线程本身就已经被挂起了，所以这里做的更像是一个复合操作——**只要前驱节点处于被取消的状态或者无法将前驱节点的状态修成Node.SIGNAL，那我们就将Node所代表的线程唤醒**，但这个条件并不意味着当前lock处于可获取的状态，有可能线程被唤醒了，但是锁还是被占有的状态，不过这样做至少是无害的，因为我们在线程被唤醒后还要去争锁，如果抢不到锁，则大不了再次被挂起。

值得注意的是，transferForSignal是有返回值的，但是我们在这个方法中并没有用到，它将在`signal()`方法中被使用。

在继续往下看`signal()`方法之前，这里我们再总结一下`signalAll()`方法：

1.  将条件队列清空（只是令`lastWaiter = firstWaiter = null`，队列中的节点和连接关系仍然还存在）
2.  将条件队列中的头节点取出，使之成为孤立节点\(`nextWaiter`,`prev`,`next`属性都为null\)
3.  如果该节点处于被Cancelled了的状态，则直接跳过该节点（由于是孤立节点，则会被GC回收）
4.  如果该节点处于正常状态，则通过enq方法将它添加到`sync queue`的末尾
5.  判断是否需要将该节点唤醒\(包括设置该节点的前驱节点的状态为SIGNAL\)，如有必要，直接唤醒该节点
6.  重复2-5，直到整个条件队列中的节点都被处理完

### signal\(\)

与`signalAll()`方法不同，`signal()`方法只会唤醒一个节点，对于AQS的实现来说，就是唤醒条件队列中第一个没有被Cancel的节点，弄懂了`signalAll()`方法，`signal()`方法就很容易理解了，因为它们大同小异：

```
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

首先依然是检查调用该方法的线程\(即当前线程\)是不是已经持有了锁，这一点和上面的`signalAll()`方法一样，所不一样的是，接下来调用的是`doSignal`方法：

```
private void doSignal(Node first) {
    do {
        // 将firstWaiter指向条件队列队头的下一个节点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 将条件队列原来的队头从条件队列中断开，则此时该节点成为一个孤立的节点
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```

这个方法也是一个`do-while`循环，目的是遍历整个条件队列，找到第一个没有被cancelled的节点，并将它添加到条件队列的末尾。如果条件队列里面已经没有节点了，则将条件队列清空（`firstWaiter=lasterWaiter=null`）。

在这里，我们用的依然用的是`transferForSignal`方法，但是用到了它的返回值，只要节点被成功添加到`sync queue`中，`transferForSignal`就返回true, 此时while循环的条件就不满足了，整个方法就结束了，即调用`signal()`方法，只会唤醒一个线程。

总结： 调用`signal()`方法会从当前条件队列中取出第一个没有被cancel的节点添加到sync队列的末尾。

### await\(\)第二部分分析

前面我们已经分析了signal方法，它会将节点添加进`sync queue`队列中，并要么立即唤醒线程，要么等待前驱节点释放锁后将自己唤醒，无论怎样，被唤醒的线程要从哪里恢复执行呢？当然是被挂起的地方呀，我们在哪里被挂起的呢？还记得吗？当然是调用了await方法的地方，以`await()`方法为例：

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 我们在这里被挂起了，被唤醒后，将从这里继续往下运行
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

这里值得注意的是，当我们被唤醒时，其实并不知道是因为什么原因被唤醒，有可能是因为其他线程调用了signal方法，也有可能是因为当前线程被中断了。

但是，无论是被中断唤醒还是被signal唤醒，被唤醒的线程最后都将离开`condition queue`，进入到`sync queue`中，这一点我们在下面分析源代码的时候详细说。

随后，线程将在`sync queue`中利用进行`acquireQueued`方法进行“**阻塞式**”争锁，抢到锁就返回，抢不到锁就继续被挂起。因此，当`await()`方法返回时，必然是保证了当前线程已经持有了lock锁。

另外有一点这里我们提前说明一下，这一点对于我们下面理解源码很重要，那就是：

> 如果从线程被唤醒，到线程获取到锁这段过程中发生过中断，该怎么处理？

我们前面分析中断的时候说过，中断对于当前线程只是个建议，由当前线程决定怎么对其做出处理。在`acquireQueued`方法中，我们对中断是不响应的，只是简单的记录抢锁过程中的中断状态，并在抢到锁后将这个中断状态返回，交于上层调用的函数处理，而这里“上层调用的函数”就是我们的`await()`方法。

那么`await()`方法是怎么对待这个中断的呢？这取决于：

> 中断发生时，线程是否已经被signal过？

如果中断发生时，当前线程并没有被signal过，则说明当前线程还处于条件队列中，属于正常在等待中的状态，此时中断将导致当前线程的正常等待行为被打断，进入到`sync queue`中抢锁，因此，在我们从await方法返回后，需要抛出`InterruptedException`，表示当前线程因为中断而被唤醒。

如果中断发生时，当前线程已经被signal过了，则说明**这个中断来的太晚了**，既然当前线程已经被signal过了，那么就说明在中断发生前，它就已经正常地被从`condition queue`中唤醒了，所以随后即使发生了中断（注意，这个中断可以发生在抢锁之前，也可以发生在抢锁的过程中），我们都将忽略它，仅仅是在`await()`方法返回后，再自我中断一下，补一下这个中断。就好像这个中断是在`await()`方法调用结束之后才发生的一样。这里之所以要“补一下”这个中断，是因为我们在用`Thread.interrupted()`方法检测是否发生中断的同时，会将中断状态清除，因此如果选择了忽略中断，则应该在`await()`方法退出后将它设成原来的样子。

关于“这个中断来的太晚了”这一点如果大家不太容易理解的话，这里打个比方：**这就好比我们去饭店吃饭，都快吃完了，有一个菜到现在还没有上，于是我们常常会把服务员叫来问：这个菜有没有在做？要是还没做我们就不要了。然后服务员会跑到厨房去问，之后跑回来说：对不起，这个菜已经下锅在炒了，请再耐心等待一下。这里，这个“这个菜我们不要了”\(发起的中断\)就来的太晚了，因为菜已经下锅了\(已经被signal过了\)。**

理清了上面的概念，我们再来看看`await()`方法是怎么做的，它用中断模式`interruptMode`这个变量记录中断事件，该变量有三个值：

 1.     `0` ： 代表整个过程中一直没有中断发生。
 2.     `THROW_IE` ： 表示退出`await()`方法时需要抛出`InterruptedException`，这种模式对应于**中断发生在signal之前**
 3.     `REINTERRUPT` ： 表示退出`await()`方法时只需要再自我中断以下，这种模式对应于**中断发生在signal之后**，即中断来的太晚了。

```
/** Mode meaning to reinterrupt on exit from wait */
private static final int REINTERRUPT =  1;
/** Mode meaning to throw InterruptedException on exit from wait */
private static final int THROW_IE    = -1;
```

接下来我们就从线程被唤醒的地方继续往下走，一步步分析源码：

#### 情况1：中断发生时，线程还没有被signal过

线程被唤醒后，我们将首先使用`checkInterruptWhileWaiting`方法检测中断的模式:

```
/**
 * Checks for interrupt, returning THROW_IE if interrupted
 * before signalled, REINTERRUPT if after signalled, or
 * 0 if not interrupted.
 */
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

这里假设已经发生过中断，则`Thread.interrupted()`方法必然返回`true`，接下来就是用`transferAfterCancelledWait`进一步判断是否发生了signal：

```
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        return true;
    }
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

上面已经说过，判断一个node是否被signal过，一个简单有效的方法就是判断它是否离开了`condition queue`, 进入到`sync queue`中。

换句话说，**只要一个节点的waitStatus还是Node.CONDITION，那就说明它还没有被signal过。**  
由于现在我们分析情况1，则当前节点的`waitStatus`必然是Node.CONDITION，则会成功执行`compareAndSetWaitStatus(node, Node.CONDITION, 0)`，将该节点的状态设置成0，然后调用`enq(node)`方法将当前节点添加进`sync queue`中，然后返回`true`。  
这里值得注意的是，我们**此时并没有断开node的nextWaiter**，所以最后一定不要忘记将这个链接断开。

再回到`transferAfterCancelledWait`调用处，可知，由于`transferAfterCancelledWait`将返回true，现在`checkInterruptWhileWaiting`将返回`THROW_IE`，这表示我们在离开await方法时应当要抛出`THROW_IE`异常。

再回到`checkInterruptWhileWaiting`的调用处：

```
public final void await() throws InterruptedException {
    /*
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    */
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); 
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 我们现在在这里！！！
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

interruptMode现在为`THROW_IE`，则我们将执行break，跳出while循环。

接下来我们将执行`acquireQueued(node, savedState)`进行争锁，注意，这里传入的需要获取锁的重入数量是`savedState`，即之前释放了多少，这里就需要再次获取多少：

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // 如果线程获取不到锁，则将在这里被阻塞住
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

`acquireQueued`我们在前面的文章中已经详细分析过了，它是一个**阻塞式**的方法，获取到锁则退出，获取不到锁则会被挂起。该方法只有在最终获取到了锁后，才会退出，并且退出时会返回当前线程的中断状态，如果我们在获取锁的过程中又被中断了，则会返回true，否则会返回false。但是其实这里返回true还是false已经不重要了，因为前面已经发生过中断了，我们就是因为中断而被唤醒的不是吗？所以无论如何，我们在退出`await()`方法时，必然会抛出`InterruptedException`。

我们这里假设它获取到了锁了，则它将回到上面的调用处，由于我们这时的interruptMode = THROW\_IE，则会跳过if语句。  
接下来我们将执行：

```
if (node.nextWaiter != null) 
    unlinkCancelledWaiters();
```

上面我们说过，当前节点的`nextWaiter`是有值的，它并没有和原来的condition队列断开，这里我们已经获取到锁了，根据独占锁的获取中的分析，我们通过`setHead`方法已经将它的`thread`属性置为null，从而将当前线程从`sync queue`"移除"了，接下来应当将它从condition队列里面移除。由于condition队列是一个单向队列，我们无法获取到它的前驱节点，所以只能从头开始遍历整个条件队列，然后找到这个节点，再移除它。

然而，事实上呢，我们并没有这么做。因为既然已经必须从头开始遍历链表了，我们就干脆一次性把链表中所有没有在等待的节点都拿出去，所以这里调用了`unlinkCancelledWaiters`方法，该方法我们在前面`await()`第一部分的分析的时候已经讲过了，它就是简单的遍历链表，找到所有`waitStatus`不为CONDITION的节点，并把它们从队列中移除。

节点被移除后，接下来就是最后一步了——汇报中断状态：

```
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

这里我们的interruptMode=THROW\_IE，说明发生了中断，则将调用`reportInterruptAfterWait`：

```
/**
 * Throws InterruptedException, reinterrupts current thread, or
 * does nothing, depending on mode.
 */
private void reportInterruptAfterWait(int interruptMode) throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

可以看出，在interruptMode=THROW\_IE时，我们就是简单的抛出了一个`InterruptedException`。

至此，情况1（中断发生于signal之前）我们就分析完了，这里我们简单总结一下：

1.  线程因为中断，从挂起的地方被唤醒
2.  随后，我们通过`transferAfterCancelledWait`确认了线程的`waitStatus`值为Node.CONDITION，说明并没有signal发生过
3.  然后我们修改线程的`waitStatus`为0，并通过`enq(node)`方法将其添加到`sync queue`中
4.  接下来线程将在`sync queue`中以阻塞的方式获取，如果获取不到锁，将会被再次挂起
5.  线程在`sync queue`中获取到锁后，将调用`unlinkCancelledWaiters`方法将自己从条件队列中移除，该方法还会顺便移除其他取消等待的锁
6.  最后我们通过`reportInterruptAfterWait`抛出了`InterruptedException`

由此可以看出，一个调用了await方法挂起的线程在被中断后不会立即抛出`InterruptedException`，而是会被添加到`sync queue`中去争锁，如果争不到，还是会被挂起；  
只有争到了锁之后，该线程才得以从`sync queue`和`condition queue`中移除，最后抛出`InterruptedException`。

所以说，**一个调用了await方法的线程，即使被中断了，它依旧还是会被阻塞住，直到它获取到锁之后才能返回，并在返回时抛出InterruptedException。中断对它意义更多的是体现在将它从condition queue中移除，加入到sync queue中去争锁，从这个层面上看，中断和signal的效果其实很像，所不同的是，在await\(\)方法返回后，如果是因为中断被唤醒，则await\(\)方法需要抛出InterruptedException异常，表示是它是被非正常唤醒的（正常唤醒是指被signal唤醒）。**

#### 情况2：中断发生时，线程已经被signal过了

这种情况对应于“中断来的太晚了”，即`REINTERRUPT`模式，我们在拿到锁退出`await()`方法后，只需要再自我中断一下，不需要抛出InterruptedException。

值得注意的是这种情况其实包含了两个子情况：

1.  被唤醒时，已经发生了中断，但此时线程已经被signal过了
2.  被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断

下面我们分别来分析：

##### 情况2.1：被唤醒时，已经发生了中断，但此时线程已经被signal过了

对于这种情况，与前面中断发生于signal之前的主要差别在于`transferAfterCancelledWait`方法：

```
final boolean transferAfterCancelledWait(Node node) {
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) { //线程A执行到这里，CAS操作将会失败
        enq(node);
        return true;
    }
    // 由于中断发生前，线程已经被signal了，则这里只需要等待线程成功进入sync queue即可
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

在这里，由于signal已经发生过了，则由我们之前分析的signal方法可知，此时当前节点的`waitStatus`必定不为Node.CONDITION，他将跳过if语句。此时当前线程可能已经在`sync queue`中，**或者正在进入到sync queue的路上**。

为什么这里会出现“正在进入到sync queue的路上”的情况呢？ 这里我们解释下：

假设当前线程为线程A, 它被唤醒之后检测到发生了中断，来到了`transferAfterCancelledWait`这里，而另一个线程B在这之前已经调用了signal方法，该方法会调用`transferForSignal`将当前线程添加到`sync queue`的末尾：

```
final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0)) // 线程B执行到这里，CAS操作将会成功
        return false; 
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

因为线程A和线程B是并发执行的，而这里我们分析的是“中断发生在signal之后”，则此时，线程B的`compareAndSetWaitStatus`先于线程A执行。这时可能出现线程B已经成功修改了node的`waitStatus`状态，但是还没来得及调用`enq(node)`方法，线程A就执行到了`transferAfterCancelledWait`方法，此时它发现`waitStatus`已经不是Condition，但是其实当前节点还没有被添加到`sync node`队列中，因此，它接下来将通过自旋，等待线程B执行完`transferForSignal`方法。

线程A在自旋过程中会不断地判断节点有没有被成功添加进`sync queue`，判断的方法就是`isOnSyncQueue`：

```
/**
 * Returns true if a node, always one that was initially placed on
 * a condition queue, is now waiting to reacquire on sync queue.
 * @param node the node
 * @return true if is reacquiring
 */
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    return findNodeFromTail(node);
}
```

该方法很好理解，只要`waitStatus`的值还为Node.CONDITION，则它一定还在condtion队列中，自然不可能在`sync`里面；而每一个调用了enq方法入队的线程：

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) { //即使这一步失败了next.prev一定是有值的
                t.next = node; // 如果t.next有值，说明上面的compareAndSetTail方法一定成功了，则当前节点成为了新的尾节点
                return t; // 返回了当前节点的前驱节点
            }
        }
    }
}
```

哪怕在设置`compareAndSetTail`这一步失败了，它的`prev`必然也是有值的，因此这两个条件只要有一个满足，就说明节点必然不在`sync queue`队列中。

另一方面，如果node.next有值，则说明它不仅在`sync queue`中，并且在它后面还有别的节点，则只要它有值，该节点必然在`sync queue`中。  
如果以上都不满足，说明这里出现了尾部分叉的情况，我们就从尾节点向前寻找这个节点：

```
/**
 * Returns true if node is on sync queue by searching backwards from tail.
 * Called only when needed by isOnSyncQueue.
 * @return true if present
 */
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

这里当然还是有可能出现从尾部反向遍历找不到的情况，但是不用担心，我们还在while循环中，无论如何，节点最后总会入队成功的。最终，`transferAfterCancelledWait`将返回false。

再回到`transferAfterCancelledWait`调用处：

```
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

则这里，由于`transferAfterCancelledWait`返回了false，则`checkInterruptWhileWaiting`方法将返回`REINTERRUPT`，这说明我们在退出该方法时只需要再次中断。

再回到`checkInterruptWhileWaiting`方法的调用处：

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) //我们在这里！！！
            break;
    }
    //当前interruptMode=REINTERRUPT，无论这里是否进入if体，该值不变
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) 
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

此时，`interruptMode`的值为`REINTERRUPT`，我们将直接跳出while循环。

接下来就和上面的情况1一样了，我们依然还是去争锁，这一步依然是阻塞式的，获取到锁则退出，获取不到锁则会被挂起。

另外由于现在`interruptMode`的值已经为`REINTERRUPT`，因此无论在争锁的过程中是否发生过中断`interruptMode`的值都还是`REINTERRUPT`。

接着就是将节点从`condition queue`中剔除，与情况1不同的是，在signal方法成功将node加入到sync queue时，该节点的nextWaiter已经是null了，所以这里这一步不需要执行。

再接下来就是报告中断状态了：

```
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

注意，这里并没有抛出中断异常，而只是将当前线程再中断一次。

至此，情况2.1（被唤醒时，已经发生了中断，但此时线程已经被signal过了）我们就分析完了，这里我们简单总结一下：

1.  线程从挂起的地方被唤醒，此时既发生过中断，又发生过signal
2.  随后，我们通过`transferAfterCancelledWait`确认了线程的`waitStatus`值已经不为Node.CONDITION，说明signal发生于中断之前
3.  然后，我们通过自旋的方式，等待signal方法执行完成，确保当前节点已经被成功添加到`sync queue`中
4.  接下来线程将在`sync queue`中以阻塞的方式获取锁，如果获取不到，将会被再次挂起
5.  最后我们通过`reportInterruptAfterWait`将当前线程再次中断，但是不会抛出`InterruptedException`

##### 情况2.2：被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断

这种情况就比上面的情况简单一点了，既然被唤醒时没有发生中断，那基本可以确信线程是被signal唤醒的，但是不要忘记还存在“假唤醒”这种情况，因此我们依然还是要检测被唤醒的原因。

那么怎么区分到底是假唤醒还是因为是被signal唤醒了呢？

如果线程是因为signal而被唤醒，则由前面分析的signal方法可知，线程最终都会离开`condition queue` 进入`sync queue`中，所以我们只需要判断被唤醒时，线程是否已经在`sync queue`中即可：

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);  // 我们在这里，线程将在这里被唤醒
        // 由于现在没有发生中断，所以interruptMode目前为0
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

线程被唤醒时，暂时还没有发生中断，所以这里interruptMode = 0, 表示没有中断发生，所以我们将继续while循环，这时我们将通过`isOnSyncQueue`方法判断当前线程是否已经在`sync queue`中了。由于已经发生过signal了，则此时node必然已经在`sync queue`中了，所以`isOnSyncQueue`将返回true,我们将退出while循环。

不过这里插一句，如果`isOnSyncQueue`检测到当前节点不在`sync queue`中，则说明既没有发生中断，也没有发生过signal，则当前线程是被“假唤醒”的，那么我们将再次进入循环体，将线程挂起。

退出while循环后接下来还是利用`acquireQueued`争锁，因为前面没有发生中断，则interruptMode=0，这时，如果在争锁的过程中发生了中断，则acquireQueued将返回true，则此时interruptMode将变为`REINTERRUPT`。

接下是判断`node.nextWaiter != null`，由于在调用signal方法时已经将节点移出了队列，所有这个条件也不成立。

最后就是汇报中断状态了，此时`interruptMode`的值为`REINTERRUPT`，说明线程在被signal后又发生了中断，这个中断发生在抢锁的过程中，这个中断来的太晚了，因此我们只是再次自我中断一下。

至此，情况2.2（被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断）我们就分析完了，这种情况和2.1很像，区别就是一个是在唤醒后就被发现已经发生了中断，一个个唤醒后没有发生中断，但是在抢锁的过成中发生了中断，**但无论如何，这两种情况都会被归结为“中断来的太晚了”**，中断模式为REINTERRUPT，情况2.2的总结如下：

1.  线程被signal方法唤醒，此时并没有发生过中断
2.  因为没有发生过中断，我们将从`checkInterruptWhileWaiting`处返回，此时interruptMode=0
3.  接下来我们回到while循环中，因为signal方法保证了将节点添加到`sync queue`中，此时while循环条件不成立，循环退出
4.  接下来线程将`在sync queue`中以阻塞的方式获取，如果获取不到锁，将会被再次挂起
5.  线程获取到锁返回后，我们检测到在获取锁的过程中发生过中断，并且此时interruptMode=0，这时，我们将interruptMode修改为REINTERRUPT
6.  最后我们通过`reportInterruptAfterWait`将当前线程再次中断，但是不会抛出`InterruptedException`

这里我们再总结以下情况2（中断发生时，线程已经被signal过了），这种情况对应于中断发生signal之后，我们**不管这个中断是在抢锁之前就已经发生了还是抢锁的过程中发生了，只要它是在signal之后发生的，我们就认为它来的太晚了，我们将忽略这个中断。因此，从await\(\)方法返回的时候，我们只会将当前线程重新中断一下，而不会抛出中断异常。**

#### 情况3： 一直没有中断发生

这种情况就更简单了，它的大体流程和上面的情况2.2差不多，只是在抢锁的过程中也没有发生异常，则interruptMode为0，没有发生过中断，因此不需要汇报中断。则线程就从`await()`方法处正常返回。

### await\(\)总结

至此，我们总算把`await()`方法完整的分析完了，这里我们对整个方法做出总结：

1.  进入`await()`时必须是已经持有了锁
2.  离开`await()`时同样必须是已经持有了锁
3.  调用`await()`会使得当前线程被封装成Node扔进条件队列，然后释放所持有的锁
4.  释放锁后，当前线程将在`condition queue`中被挂起，等待signal或者中断
5.  线程被唤醒后会将会离开`condition queue`进入`sync queue`中进行抢锁
6.  若在线程抢到锁之前发生过中断，则根据中断发生在signal之前还是之后记录中断模式
7.  线程在抢到锁后进行善后工作（离开condition queue, 处理中断异常）
8.  线程已经持有了锁，从`await()`方法返回

![await() 方法流程](https://image-static.segmentfault.com/213/804/2138040078-5ba291924c178_articlex "await() 方法流程")

在这一过程中我们尤其要关注中断，如前面所说，**中断和signal所起到的作用都是将线程从`condition queue`中移除，加入到`sync queue`中去争锁，所不同的是，signal方法被认为是正常唤醒线程，中断方法被认为是非正常唤醒线程，如果中断发生在signal之前，则我们在最终返回时，应当抛出`InterruptedException`；如果中断发生在signal之后，我们就认为线程本身已经被正常唤醒了，这个中断来的太晚了，我们直接忽略它，并在`await()`返回时再自我中断一下，这种做法相当于将中断推迟至`await()`返回时再发生。**

### awaitUninterruptibly\(\)

在前面我们分析的`await()`方法中，中断起到了和signal同样的效果，但是中断属于将一个等待中的线程非正常唤醒，可能即使线程被唤醒后，也抢到了锁，但是却发现当前的等待条件并没有满足，则还是得把线程挂起。因此我们有时候并不希望await方法被中断，`awaitUninterruptibly()`方法即实现了这个功能：

```
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true; // 发生了中断后线程依旧留在了condition queue中，将会再次被挂起
    }
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

首先，从方法签名上就可以看出，这个方法不会抛出中断异常，我们拿它和`await()`方法对比一下：

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())  // 不同之处
        throw new InterruptedException(); // 不同之处
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;  // 不同之处
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)  // 不同之处
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)  // 不同之处
        interruptMode = REINTERRUPT;  // 不同之处
    if (node.nextWaiter != null)  // 不同之处
        unlinkCancelledWaiters(); // 不同之处
    if (interruptMode != 0) // 不同之处
        reportInterruptAfterWait(interruptMode); // 不同之处
}
```

由此可见，`awaitUninterruptibly()`全程忽略中断，即使是当前线程因为中断被唤醒，该方法也只是简单的记录中断状态，然后再次被挂起（因为并没有并没有任何操作将它添加到`sync queue`中）

要使当前线程离开`condition queue`去争锁，则**必须是**发生了signal事件。

最后，当线程在获取锁的过程中发生了中断，该方法也是不响应，只是在最终获取到锁返回时，再自我中断一下。可以看出，该方法和“中断发生于signal之后的”`REINTERRUPT`模式的`await()`方法很像。

至此，该方法我们就分析完了，如果你之前`await()`方法已经弄懂了，这个`awaitUninterruptibly()`方法就很容易理解了。它的核心思想是：

1.  中断虽然会唤醒线程，但是不会导致线程离开`condition queue`，如果线程只是因为中断而被唤醒，则他将再次被挂起
2.  只有signal方法会使得线程离开`condition queue`
3.  调用该方法时或者调用过程中如果发生了中断，仅仅会在该方法结束时再自我中断以下，不会抛出`InterruptedException`

### awaitNanos\(long nanosTimeout\)

前面我们看的方法，无论是`await()`还是`awaitUninterruptibly()`，它们在抢锁的过程中都是阻塞式的，即一直到抢到了锁才能返回，否则线程还是会被挂起，这样带来一个问题就是线程如果长时间抢不到锁，就会一直被阻塞，因此我们有时候更需要带超时机制的抢锁，这一点和带超时机制的`wait(long timeout)`是很像的，我们直接来看源码：

```
public final long awaitNanos(long nanosTimeout) throws InterruptedException {
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);*/
    final long deadline = System.nanoTime() + nanosTimeout;
    /*int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);
            break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
        nanosTimeout = deadline - System.nanoTime();
    }
    /*if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    return deadline - System.nanoTime();
}
```

该方法几乎和`await()`方法一样，只是多了超时时间的处理，我们上面已经把和`await()`方法相同的部分注释起来了，只留下了不同的部分，这样它们的区别就变得更明显了。

该方法的主要设计思想是，如果设定的超时时间还没到，我们就将线程挂起；超过等待的时间了，我们就将线程从`condtion queue`转移到`sync queue`中。注意这里对于超时时间有一个小小的优化——当设定的超时时间很短时（小于`spinForTimeoutThreshold`的值），我们就是简单的自旋，而不是将线程挂起，以减少挂起线程和唤醒线程所带来的时间消耗。

不过这里还有一处值得注意，就是`awaitNanos(0)`的意义，我们在wait, notify, notifyAll曾经提到过，`wait(0)`的含义是无限期等待，而我们在`awaitNanos(long nanosTimeout)`方法中是怎么处理`awaitNanos(0)`的呢？

```
if (nanosTimeout <= 0L) {
    transferAfterCancelledWait(node);
    break;
}
```

从这里可以看出，如果设置的等待时间本身就小于等于0，当前线程是会直接从`condition queue`中转移到`sync queue`中的，并不会被挂起，也不需要等待signal，这一点确实是更复合逻辑。如果需要线程只有在signal发生的条件下才会被唤醒，则应该用上面的`awaitUninterruptibly()`方法。

### await\(long time, TimeUnit unit\)

看完`awaitNanos(long nanosTimeout)`再看`await(long time, TimeUnit unit)`方法就更简单了，它就是在`awaitNanos(long nanosTimeout)`的基础上多了对于超时时间的时间单位的设置，但是在内部实现上还是会把时间转成纳秒去执行，这里我们直接拿它和上面的`awaitNanos(long nanosTimeout)`方法进行对比，只给出不同的部分：

```
public final boolean await(long time, TimeUnit unit) throws InterruptedException {
    long nanosTimeout = unit.toNanos(time);
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;*/
    /*boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {*/
            timedout = transferAfterCancelledWait(node);
            /*break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    return !timedout;
}
```

可以看出，这两个方法主要的差别就体现在返回值上面，`awaitNanos(long nanosTimeout)`的返回值是剩余的超时时间，如果该值大于0，说明超时时间还没到，则说明该返回是由signal行为导致的，而`await(long time, TimeUnit unit)`的返回值就是`transferAfterCancelledWait(node)`的值，我们知道，如果调用该方法时，node还没有被signal过则返回true，node已经被signal过了，则返回false。因此当`await(long time, TimeUnit unit)`方法返回true，则说明在超时时间到之前就已经发生过signal了，该方法的返回是由signal方法导致的而不是超时时间。

综上，调用`await(long time, TimeUnit unit)`其实就等价于调用`awaitNanos(unit.toNanos(time)) > 0` 方法，关于这一点，我们在介绍condition接口的时候也已经提过了。

### awaitUntil\(Date deadline\)

`awaitUntil(Date deadline)`方法与上面的几种带超时的方法也基本类似，所不同的是它的超时时间是一个绝对的时间，我们直接拿它来和上面的`await(long time, TimeUnit unit)`方法对比：

```
public final boolean awaitUntil(Date deadline) throws InterruptedException {
    long abstime = deadline.getTime();
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        if (System.currentTimeMillis() > abstime) {
            /*timedout = transferAfterCancelledWait(node);
            break;
        }*/
        LockSupport.parkUntil(this, abstime);
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
    }
    /*
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;*/
}
```

可见，这里大段的代码都是重复的，区别就是在超时时间的判断上使用了绝对时间，其实这里的deadline就和`awaitNanos(long nanosTimeout)`以及`await(long time, TimeUnit unit)`内部的`deadline`变量是等价的，另外就是在这个方法中，没有使用`spinForTimeoutThreshold`进行自旋优化，因为一般调用这个方法，目的就是设定一个较长的等待时间，否则使用上面的相对时间会更方便一点。

至此，AQS对于Condition接口的实现我们就全部分析完了。
