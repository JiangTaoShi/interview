## 继承关系

ReadLock和WriteLock是ReentrantReadWriteLock的两个内部类，Lock的上锁和释放锁都是通过AQS来实现的。

![](https://img-blog.csdnimg.cn/20210131225330914.png)

AQS定义了独占模式的acquire\(\)和release\(\)方法，共享模式的acquireShared\(\)和releaseShared\(\)方法。

还定义了抽象方法tryAcquire\(\)、tryAcquiredShared\(\)、tryRelease\(\)和tryReleaseShared\(\)由子类实现。

tryAcquire\(\)和tryAcquiredShared\(\)分别对应独占模式和共享模式下的锁的尝试获取，就是通过这两个方法来实现公平性和非公平性。

在尝试获取中，如果新来的线程必须先入队才能获取锁就是公平的，否则就是非公平的。这里可以看出AQS定义整体的同步器框架，具体实现放手交由子类实现。

## 源码分析

ReadLock和WriteLock方法都是通过调用Sync的方法实现的，所以我们先来分析一下Sync源码：

AQS 的状态state是32位（int 类型）的，辦成两份。

读锁用高16位，表示持有读锁的线程数（sharedCount），写锁低16位，表示写锁的重入次数 （exclusiveCount）。

状态值为 0 表示锁空闲，sharedCount不为 0 表示分配了读锁，exclusiveCount 不为 0 表示分配了写锁。

sharedCount和exclusiveCount 一般不会同时不为 0，只有当线程占用了写锁，该线程可以重入获取读锁，反之不成立。

![](https://img-blog.csdnimg.cn/20210131225352400.png)

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
  
       static final int SHARED_SHIFT   = 16;
       // 由于读锁用高位部分，所以读锁个数加1，其实是状态值加 2^16
       static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
       // 写锁的可重入的最大次数、读锁允许的最大数量
       static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
       // 写锁的掩码，用于状态的低16位有效值
       static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
       // 读锁计数，当前持有读锁的线程数
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    // 写锁的计数，也就是它的重入次数
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
}
```

重入计数：

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    /**
     * 每个线程特定的 read 持有计数。存放在ThreadLocal，不需要是线程安全的。
     */
    static final class HoldCounter {
        int count = 0;
        // 使用id而不是引用是为了避免保留垃圾。注意这是个常量。
        final long tid = Thread.currentThread().getId();
    }
    /**
     * 采用继承是为了重写 initialValue 方法，这样就不用进行这样的处理：
     * 如果ThreadLocal没有当前线程的计数，则new一个，再放进ThreadLocal里。
     * 可以直接调用 get。
     * */
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }
    /**
     * 保存当前线程重入读锁的次数的容器。在读锁重入次数为 0 时移除。
     */
    private transient ThreadLocalHoldCounter readHolds;
    /**
     * 最近一个成功获取读锁的线程的计数。这省却了ThreadLocal查找，
     * 通常情况下，下一个释放线程是最后一个获取线程。这不是 volatile 的，
     * 因为它仅用于试探的，线程进行缓存也是可以的
     * （因为判断是否是当前线程是通过线程id来比较的）。
     */
    private transient HoldCounter cachedHoldCounter;
    /**
     * firstReader是这样一个特殊线程：它是最后一个把 共享计数 从 0 改为 1 的
     * （在锁空闲的时候），而且从那之后还没有释放读锁的。如果不存在则为null。
     * firstReaderHoldCount 是 firstReader 的重入计数。
     *
     * firstReader 不能导致保留垃圾，因此在 tryReleaseShared 里设置为null，
     * 除非线程异常终止，没有释放读锁。
     *
     * 作用是在跟踪无竞争的读锁计数时非常便宜。
     *
     * firstReader及其计数firstReaderHoldCount是不会放入 readHolds 的。
     */
    private transient Thread firstReader = null;
    private transient int firstReaderHoldCount;
    Sync() {
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // 确保 readHolds 的内存可见性，利用 volatile 写的内存语义。
    }
}
```

Sync中提供了很多方法，但是有两个方法是抽象的，子类必须实现。下面以FairSync为例，分析一下这两个抽象方法：

```
static final class FairSync extends Sync {
        private static final long serialVersionUID = -2274990926593161451L;
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }
```

writerShouldBlock和readerShouldBlock方法都表示当有别的线程也在尝试获取锁时，是否应该阻塞。  

对于公平模式，hasQueuedPredecessors\(\)方法表示前面是否有等待线程。一旦前面有等待线程，那么为了遵循公平，当前线程也就应该被挂起。

下面再来看NonfairSync的实现：

```
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = -8159625535654395037L;
        final boolean writerShouldBlock() {
            return false; // 写线程总是可以闯入
        }
        final boolean readerShouldBlock() {
           //
            return apparentlyFirstQueuedIsExclusive();
        }
    }
/**如果头节点的下一个节点是独占线程，为了防止独占线程也就是写线程饥饿等待，则后入线程应该排队，否则可以闯入*/
final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
```

从上面可以看到，非公平模式下，writerShouldBlock直接返回false，说明不需要阻塞；

而readShouldBlock调用了apparentFirstQueuedIsExcluisve\(\)方法。

如果等待队列中第一个等待线程想获取写锁，返回true；否则返回false。

也就说明，如果等待队列中第一个等待线程想获取写锁，那么该读线程应该阻塞。

如果当前全局处于读锁状态，且等待队列中第一个等待线程想获取写锁。

那么当前线程能够获取到读锁的条件为：当前线程获取了写锁，还未释放；当前线程获取了读锁，这一次只是重入读锁而已；其它情况当前线程入队尾。

之所以这样处理一方面是为了效率，一方面是为了避免想获取写锁的线程饥饿，老是得不到执行的机会 。

例如：线程C请求一个写锁，由于当前其他两个线程拥有读锁，写锁获取失败，线程C入队列\(根据规则i\)，如下所示

![](https://img-blog.csdnimg.cn/20210131230651637.png)

AQS初始化会创建一个空的头节点，C入队列，然后会休眠，等待其他线程释放锁唤醒。

此时线程D也来了，线程D想获取一个读锁，上面规则，队列中第一个等待线程C请求的是写锁，为避免写锁迟迟获取不到，并且线程D不是重入获取读锁，所以线程D也入队，如下图所示：

![](https://img-blog.csdnimg.cn/20210131230714230.png)

## 读锁获取

### 获取共享lock 方法 acquireShared

```java
public final void acquireShared(int arg){
    if(tryAcquireShared(arg) < 0){  // 1. 调用子类, 获取共享 lock  返回 < 0, 表示失败
        doAcquireShared(arg);       // 2. 调用 doAcquireShared 当前 线程加入 Sync Queue 里面, 等待获取 lock
    }
}
```

Sync实现的尝试获取锁

在以下几种情况，获取读锁会失败：

（1）有线程持有写锁，且该线程不是当前线程，获取锁失败。

（2）写锁空闲 且 公平策略决定 读线程应当被阻塞，除了重入获取，其他获取锁失败。

（3）读锁数量达到最多，抛出异常。

除了以上三种情况，该线程会循环尝试获取读锁直到成功。

```java
protected final int tryAcquireShared(int unused) {
  Thread current = Thread.currentThread();
  int c = getState();
  if (exclusiveCount(c) != 0 &&
      getExclusiveOwnerThread() != current)
      return -1;            //1.有线程持有写锁，且该线程不是当前线程，获取锁失败
  int r = sharedCount(c);   //2.获取读锁计数
  if (!readerShouldBlock() &&
      r < MAX_COUNT &&
      compareAndSetState(c, c + SHARED_UNIT)) {//3.如果不应该阻塞，且读锁数<MAX_COUNT且设置同步状态state成功，获取锁成功。
      if (r == 0) {            //下面对firstReader的处理：firstReader是不会放到readHolds里的，这样，在读锁只有一个的情况下，就避免了查找readHolds。
          firstReader = current;
          firstReaderHoldCount = 1;
      } else if (firstReader == current) {
          firstReaderHoldCount++;
      } else {
//  // 非 firstReader 读锁重入计数更新
          HoldCounter rh = cachedHoldCounter;
          if (rh == null || rh.tid != current.getId())
              cachedHoldCounter = rh = readHolds.get();
          else if (rh.count == 0)
              readHolds.set(rh);
          rh.count++;
      }
      return 1;
  }
      //4.获取读锁失败，放到循环里重试。
  return fullTryAcquireShared(current);
}
```

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;     //1.有线程持有写锁，且该线程不是当前线程，获取锁失败
            //2.有线程持有写锁，且该线程是当前线程，则应该放行让其重入获取锁，否则会造成死锁。
        } else if (readerShouldBlock()) {
            //3.写锁空闲  且  公平策略决定 读线程应当被阻塞
              // 下面的处理是说，如果是已获取读锁的线程重入读锁时，
              // 即使公平策略指示应当阻塞也不会阻塞。
              // 否则，这也会导致死锁的。
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId()) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                //4.需要阻塞且是非重入(还未获取读锁的)，获取失败。
                if (rh.count == 0)
                    return -1;
            }

        }
        //5.写锁空闲  且  公平策略决定线程可以获取读锁
        if (sharedCount(c) == MAX_COUNT)//6.读锁数量达到最多
            throw new Error("Maximum lock count exceeded");
        //7. 申请读锁成功，下面的处理跟tryAcquireShared是类似的。
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

### 获取共享lock 方法 doAcquireShared

```java
private void doAcquireShared(int arg){
    final Node node = addWaiter(Node.SHARED);       // 1. 将当前的线程封装成 Node 加入到 Sync Queue 里面
    boolean failed = true;

    try {
        boolean interrupted = false;
        for(;;){
            final Node p = node.predecessor();      // 2. 获取当前节点的前继节点 (当一个n在 Sync Queue 里面, 并且没有获取 lock 的 node 的前继节点不可能是 null)
            if(p == head){
                int r = tryAcquireShared(arg);      // 3. 判断前继节点是否是head节点(前继节点是head, 存在两种情况 (1) 前继节点现在占用 lock (2)前继节点是个空节点, 已经释放 lock, node 现在有机会获取 lock); 则再次调用 tryAcquireShared 尝试获取一下
                if(r >= 0){
                    setHeadAndPropagate(node, r);   // 4. 获取 lock 成功, 设置新的 head, 并唤醒后继获取  readLock 的节点
                    p.next = null; // help GC
                    if(interrupted){               // 5. 在获取 lock 时, 被中断过, 则自己再自我中断一下(外面的函数可能需要这个参数)
                        selfInterrupt();
                    }
                    failed = false;
                    return;
                }
            }

            if(shouldParkAfterFailedAcquire(p, node) && // 6. 调用 shouldParkAfterFailedAcquire 判断是否需要中断(这里可能会一开始 返回 false, 但在此进去后直接返回 true(主要和前继节点的状态是否是 signal))
                    parkAndCheckInterrupt()){           // 7. 现在lock还是被其他线程占用 那就睡一会, 返回值判断是否这次线程的唤醒是被中断唤醒
                interrupted = true;
            }
        }
    }finally {
        if(failed){             // 8. 在整个获取中出错(比如线程中断/超时)
            cancelAcquire(node);  // 9. 清除 node 节点(清除的过程是先给 node 打上 CANCELLED标志, 然后再删除)
        }
    }
}
```

独占锁模式获取成功以后设置头结点然后返回中断状态，结束流程。而共享锁模式获取成功以后，调用了setHeadAndPropagate方法，从方法名就可以看出除了设置新的头结点以外还有一个传递动作，一起看下代码：

```java
//两个入参，一个是当前成功获取共享锁的节点，一个就是tryAcquireShared方法的返回值，注意上面说的，它可能大于0也可能等于0
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; //记录当前头节点
    //设置新的头节点，即把当前获取到锁的节点设置为头节点
    //注：这里是获取到锁之后的操作，不需要并发控制
    setHead(node);
    //这里意思有两种情况是需要执行唤醒操作
    //1.propagate > 0 表示调用方指明了后继节点有可能需要被唤醒，因为此方法是获取读锁过程调用，那么后面节点很可能也要获取读锁
    //2.头节点后面的节点需要被唤醒（waitStatus<0），不论是老的头结点还是新的头结点
    if (propagate > 0 || h == null || h.waitStatus < 0) {
        Node s = node.next;
        //如果当前节点的后继节点是共享类型获取没有后继节点，则进行唤醒
        //这里可以理解为除非明确指明不需要唤醒（后继等待节点是独占类型），否则都要唤醒
        //这里的初衷是   后一个节点正好是共享节点，就唤醒，实现共享，独占有锁释放时候唤醒
        if (s == null || s.isShared())
            //后面详细说
            doReleaseShared();
    }
}

private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

**注：这个唤醒操作在releaseShared\(\)方法里也会调用。唤醒后面想获取锁的节点。**

```java
private void doReleaseShared() {
    for (;;) {
    //唤醒操作由头结点开始，注意这里的头节点已经是上面新设置的头结点了
    //其实就是唤醒上面新获取到共享锁的节点的后继节点
    Node h = head;
    if (h != null && h != tail) {
        int ws = h.waitStatus;
        //表示后继节点需要被唤醒
        if (ws == Node.SIGNAL) {
            //这里需要控制并发，因为入口有setHeadAndPropagate跟releaseShared两个，避免两次unpark
            if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                continue;      
            //执行唤醒操作      
            unparkSuccessor(h);
        }
        //如果后继节点暂时不需要唤醒，则把当前节点状态设置为PROPAGATE确保以后可以传递下去
        else if (ws == 0 &&
                 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
            continue;                
    }
    //如果头结点没有发生变化，表示设置完成，退出循环
    //如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试
    if (h == head)                   
        break;
}
}
```

这里分析一下共享锁是如何进行传递的

### 读锁的释放

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

释放锁tryReleaseShared由子类Sync实现

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 清理firstReader缓存 或 readHolds里的重入计数
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != current.getId())
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 完全释放读锁
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count; // 主要用于重入退出
    }
    // 循环在CAS更新状态值，主要是把读锁数量减 1
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // 释放读锁对其他读线程没有任何影响，
            // 但可以允许等待的写线程继续，如果读锁、写锁都空闲。
            return nextc == 0;
    }
}
```

### 写锁的获取

写锁的获取和ReentrantLock独占锁的锁获取过程几乎一样，除了tryAcquire（）方法，要考虑读锁的情况。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

在以下情况，写锁获取失败：

（1） 写锁为0，读锁不为0 或者写锁不为0，且当前线程不是已获取独占锁的线程，锁获取失败。

（2）写锁数量已达到最大值，写锁获取失败。

（3）当前线程应该阻塞，或者设置同步状态state失败，获取锁失败。

```java
protected final boolean tryAcquire(int acquires) {

    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // 1.写锁为0，读锁不为0    或者写锁不为0，且当前线程不是已获取独占锁的线程，锁获取失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //2. 写锁数量已达到最大值，写锁获取失败
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    //3.当前线程应该阻塞，或者设置同步状态state失败，获取锁失败。
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

### 写锁的释放

```java
public final boolean release(int arg) {
if (tryRelease(arg)) {
    Node h = head;
    if (h != null && h.waitStatus != 0)
        unparkSuccessor(h);
    return true;
}
return false;
}

protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

### 总结：

（1）首先说一下公平锁和非公平锁的区别，

公平锁：当线程发现已经有线程在排对获取锁了，那么它必须排队，除了一种情况就是，线程已经占有锁，此次是重入，不用排队。

非公平锁：只有一种情况需排队，其他情况不用排队就可以尝试获取锁： 如果当前全局处于读锁状态，且等待队列中第一个等待线程想获取写锁，那么当前线程能够获取到读锁的条件为：当前线程获取了写锁，还未释放；当前线程获取了读锁，这一次只是重入读锁而已；其它情况当前线程入队尾。

**（2）获取读锁和释放读锁**

获取锁的过程：

1.  当线程调用acquireShared\(\)申请获取锁资源时，如果成功，则进入临界区。

2.  当获取锁失败时，则创建一个共享类型的节点并进入一个FIFO等待队列，然后被挂起等待唤醒。

3.  当队列中的等待线程被唤醒以后就重新尝试获取锁资源，如果成功则**唤醒后面还在等待的共享节点并把该唤醒事件传递下去，即会依次唤醒在该节点后面的所有共享节点**，然后进入临界区，否则继续挂起等待。

**释放锁过程：**

1.  当线程调用releaseShared\(\)进行锁资源释放时，如果释放成功，则唤醒队列中等待的节点，如果有的话。

**（3）跟独占锁相比**，共享锁的主要特征在于当一个在等待队列中的共享节点成功获取到锁以后（它获取到的是共享锁），既然是共享，那它必须要依次唤醒后面所有可以跟它一起共享当前锁资源的节点，毫无疑问，这些节点必须也是在等待共享锁（这是大前提，如果等待的是独占锁，那前面已经有一个共享节点获取锁了，它肯定是获取不到的）。当共享锁被释放的时候，可以用读写锁为例进行思考，当一个读锁被释放，此时不论是读锁还是写锁都是可以竞争资源的。
