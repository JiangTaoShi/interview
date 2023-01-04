## 前言

在前面两篇系列文章中，已经讲解了独占锁的获取和释放过程，而共享锁的获取与释放过程也很类似，如果你前面独占锁的内容都看懂了，那么共享锁你也就触类旁通了。

## 共享锁与独占锁的区别

共享锁与独占锁最大的区别在于，共享锁的函数名里面都带有一个`Shared`（抖个机灵，当然不是这个）。

- 独占锁是线程独占的，同一时刻只有一个线程能拥有独占锁，AQS里将这个线程放置到`exclusiveOwnerThread`成员上去。
- 共享锁是线程共享的，同一时刻能有多个线程拥有共享锁，但AQS里并没有用来存储获得共享锁的多个线程的成员。
- 如果一个线程刚获取了共享锁，那么在其之后等待的线程也很有可能能够获取到锁。但独占锁不会这样做，因为锁是独占的。
- 当然，如果一个线程刚释放了锁，不管是独占锁还是共享锁，都需要唤醒在后面等待的线程。

让我们把共享锁与独占锁的函数名都列出来看一下：

| 独占锁 | 共享锁 |
| :-: | :-: |
| `tryAcquire(int arg)` | `tryAcquireShared(int arg)` |
| `tryAcquireNanos(int arg, long nanosTimeout)` | `tryAcquireSharedNanos(int arg, long nanosTimeout)` |
| `acquire(int arg)` | `acquireShared(int arg)` |
| `acquireQueued(final Node node, int arg)` | `doAcquireShared(int arg)` |
| `acquireInterruptibly(int arg)` | `acquireSharedInterruptibly(int arg)` |
| `doAcquireInterruptibly(int arg)` | `doAcquireSharedInterruptibly(int arg)` |
| `doAcquireNanos(int arg, long nanosTimeout)` | `doAcquireSharedNanos(int arg, long nanosTimeout)` |
| `release(int arg)` | `releaseShared(int arg)` |
| `tryRelease(int arg)` | `tryReleaseShared(int arg)` |
| \- | `doReleaseShared()` |

从上表可以看到，共享锁的函数是和独占锁是一一对应的，而且大部分只是函数名加了个`Shared`，从逻辑上看也是很相近的。

而`doReleaseShared`没有对应到独占锁的方法是因为它的逻辑是包含了`unparkSuccessor`，是建立在`unparkSuccessor`之上的，你可以简单地认为，`doReleaseShared`对应到独占锁的方法是`unparkSuccessor`。最主要的是，它们的使用时机不同：

- 在独占锁中，释放锁时，会调用`unparkSuccessor`。
- 在共享锁中，获得锁和释放锁时，都会调用到`doReleaseShared`。不过获得共享锁时，是在一定条件下调用`doReleaseShared`。

# 观察Semaphore的内部类

为了看到AQS的子类实现部分，我们从Semaphore看起。

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        Sync(int permits) {
            setState(permits);
        }

        final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
	}

    static final class NonfairSync extends Sync {
        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
    }

    static final class FairSync extends Sync {
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```

- 首先看到`Sync`的构造器，看来参数`permits`是代表共享锁的数量。
- 观察`tryAcquireShared`的公平和非公平锁的逻辑，发现区别只是 公平锁里面每次循环都会判断`hasQueuedPredecessors()`的返回值。

这里先给大家讲一下`tryAcquireShared`：  
参数`acquires`代表这次想要获得的共享锁的数量是多少。  
返回值则有三种情况：

1.  如果返回值大于0，说明获取共享锁成功，并且后续获取也可能获取成功。
2.  如果返回值等于0，说明获取共享锁成功，但后续获取可能不会成功。
3.  如果返回值小于0，说明获取共享锁失败。

直接看公平版本的`tryAcquireShared`，上面返回的地方：

- `hasQueuedPredecessors()`如果返回了true，说明有线程排在了当前线程之前，现在公平版本又不能插队，所以结束返回-1，代表获取失败。
- 如果`remaining < 0`成立，说明想要获取的共享锁数量已经超过了当前已有的数量，那么直接返回一个负数`remaining`，代表获取失败。
- 如果`remaining < 0`不成立，说明想要获取的共享锁数量没有超过了当前已有的数量（等于0代表将会获取剩余所有的共享锁）。且接下来如果`compareAndSetState(available, remaining)`成功，那么返回一个`>=0`的数`remaining`，代表获取成功。

接下来我们谈谈共享锁的`tryAcquireShared`和独占锁的`tryAcquire`的不同之处：

 -    `tryAcquire`的返回值是boolean型，它只代表两种状态（获取成功或失败）。而`tryAcquireShared`的返回值是int型，如上有三种情况。
 -    `tryAcquireShared`使用了自旋（死循环），但`tryAcquire`没有自旋。这将导致`tryAcquire`最多执行一次CAS操作修改同步器状态，但`tryAcquireShared`可能有多次。`tryAcquireShared`具体地讲，只要`remaining`是`>=0`的（`remaining < 0`不成立），就一定会去尝试CAS设置同步器的状态。使用自旋的原因想必是，锁是共享的，既然还可能获取到（`remaining`是`>=0`的），就一定要去尝试。

```java
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }
```

最后再看`tryReleaseShared`的实现，也用到了自旋操作，因为完全有可能多个线程同时释放共享锁，同时调用`tryReleaseShared`，所以需要用自旋保证 共享锁的释放最终能体现到同步器的状态上去。另外，除非int型溢出，那么此函数只可能返回true。

# 共享锁的获取

上面讲完了Semaphore的内部类，接下来我们就可以尽情地在AQS的源码里畅游了。

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

`acquireShared`对应到独占锁的方法是`acquire`：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

咋一看感觉差别有点大，其实我们被迷惑了，后面我们会发现，之所以`acquireShared`里没有显式调用`addWaiter`和`selfInterrupt`，是因为这两件事都被放到了`doAcquireShared(arg)`的逻辑里面了。

接下来看看`doAcquireShared`方法的逻辑，它对应到独占锁是`acquireQueued`，除了上面提到的两件事，它们其实差别很少：

```java
    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED); //这件事放到里面来了
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {  //前驱是head时，才尝试获得共享锁
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {  //获取共享锁成功时，才进行善后操作
                        setHeadAndPropagate(node, r);  //独占锁这里调用的是setHead
                        p.next = null; 
                        if (interrupted)
                            selfInterrupt(); //这件事也放到里面来了
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

而`acquireQueued`在获得独占锁成功时，执行的是：

```java
if (p == head && tryAcquire(arg)) {  // tryAcquire返回true，代表获取独占锁成功
    setHead(node);
    p.next = null; 
    failed = false;
    return interrupted;
}
```

所以对比发现，共享锁的`doAcquireShared`有两处不同：

 1.     **创建的节点不同**。共享锁使用`addWaiter(Node.SHARED)`，所以会创建出想要获取共享锁的节点。而独占锁使用`addWaiter(Node.EXCLUSIVE)`。
 2.     **获取锁成功后的善后操作不同**。共享锁使用`setHeadAndPropagate(node, r)`，因为刚获取共享锁成功后，后面的线程也有可能成功获取，所以需要在一定条件唤醒head后继。而独占锁使用`setHead(node)`。

```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; 
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }
```

`setHead`函数只是将刚成为将成为head的节点变成一个dummy node。而`setHeadAndPropagate`里也会调用`setHead`函数。但是它在一定条件下还可能会调用`doReleaseShared`，看来这就是单词`Propagate`的由来了，也就是我们一直说的“如果一个线程刚获取了共享锁，那么在其之后等待的线程也很有可能能够获取到锁”。

`doReleaseShared`留到之后讲解，因为共享锁的释放也会用到它。

# 共享锁的释放

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

`releaseShared`对应到独占锁的方法是`release`：

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
```

可见独占锁的逻辑比较简单，只是在head状态不为0时，就唤醒head后继。

而共享锁的逻辑则直接调用了`doReleaseShared`，但在获取共享锁成功时，也可能会调用到`doReleaseShared`。也就是说，获取共享锁的线程（分为：已经获取到的线程 即执行`setHeadAndPropagate`中、等待获取中的线程 即阻塞在`shouldParkAfterFailedAcquire`里）和释放共享锁的线程 可能在同时执行这个`doReleaseShared`。

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

我们来仔细分析下这个函数的逻辑：

- 逻辑是一个死循环，每次循环中重新读取一次head，然后保存在局部变量`h`中，再配合`if(h == head) break;`，这样，循环检测到head没有变化时就会退出循环。注意，head变化一定是因为：acquire thread被唤醒，之后它成功获取锁，然后setHead设置了新head。而且注意，只有通过`if(h == head) break;`即head不变才能退出循环，不然会执行多次循环。
- `if (h != null && h != tail)`判断队列是否至少有两个node，如果队列从来没有初始化过（head为null），或者head就是tail，那么中间逻辑直接不走，直接判断head是否变化了。
- 如果队列中有两个或以上个node，那么检查局部变量`h`的状态：
  - 如果状态为SIGNAL，说明h的后继是需要被通知的。通过对CAS操作结果取反，将`compareAndSetWaitStatus(h, Node.SIGNAL, 0)`和`unparkSuccessor(h)`绑定在了一起。说明了只要head成功得从SIGNAL修改为0，那么head的后继的代表线程肯定会被唤醒了。
  - 如果状态为0，说明h的后继所代表的线程已经被唤醒或即将被唤醒，并且这个中间状态即将消失，要么由于acquire thread获取锁失败再次设置head为SIGNAL并再次阻塞，要么由于acquire thread获取锁成功而将自己（head后继）设置为新head并且只要head后继不是队尾，那么新head肯定为SIGNAL。所以设置这种中间状态的head的status为PROPAGATE，让其status又变成负数，这样可能被 被唤醒线程（因为正常来讲，被唤醒线程的前驱，也就是head会被设置为0的，所以被唤醒线程发现head不为0，就会知道自己应该去唤醒自己的后继了） 检测到。
  - 如果状态为PROPAGATE，直接判断head是否变化。
- 两个continue保证了进入那两个分支后，只有当CAS操作成功后，才可能去执行`if(h == head) break;`，才可能退出循环。
- `if(h == head) break;`保证了，只要在某个循环的过程中有线程刚获取了锁且设置了新head，就会再次循环。目的当然是为了再次执行`unparkSuccessor(h)`，即唤醒队列中第一个等待的线程。

## head状态为0的情况

- 如果等待队列中只有一个dummy node（它的状态为0），那么head也是tail，且head的状态为0。
- 等待队列中当前只有一个dummy node（它的状态为0），acquire thread获取锁失败了（无论独占还是共享），将当前线程包装成node放到队列中，**此时队列中有两个node**，但当前线程还没来得及执行一次`shouldParkAfterFailedAcquire`。
- **此时队列中有多个node**，有线程刚释放了锁，刚执行了`unparkSuccessor`里的`if (ws < 0) compareAndSetWaitStatus(node, ws, 0);`把head的状态设置为了0，然后唤醒head后继线程，head后继线程获取锁成功，直到head后继线程将自己设置为AQS的新head的这段时间里，head的状态为0。
  - 具体地讲，如果是共享锁的话，一定是在调用`unparkSuccessor`之前就把head的状态变成0了，因为`if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))`。
  - 上面这种情况还可以继续延伸，在“唤醒head后继线程”后，head后继线程唤醒后第一次循环获取锁失败（你可能会疑问，上面的场景明明是刚有人释放了锁，为什么这里会失败，因为多线程环境下有可能被别的不公平获取方式插队了），调用`shouldParkAfterFailedAcquire`又将head设置回SIGNAL了，然后第二次循环开始之前（假设head后继线程此时分出去时间片），又有一个释放锁的线程在执行`doReleaseShared`里面的`compareAndSetWaitStatus(h, Node.SIGNAL, 0)`成功并且还unpark了处于唤醒状态的head后继线程，然后第二次循环开始（假设head后继线程此时得到时间片），获取锁成功。
    - 注意，如果unpark一个已经唤醒的线程，它的副作用是下一次park这个线程，线程不会阻塞。下下次park线程，才会阻塞。

总结：

- head状态为0的情况，属于一种中间状态。
- 这种中间状态将变化为，head状态为SIGNAL，不管acquire thread接下来是获取锁成功还是失败。不过获取锁成功这种情况，需要考虑head后继（也就是包装acquire thread的那个node）不是队尾，如果是队尾，那么新head的状态也是为0的了。

## 同时执行doReleaseShared

这个函数的难点在于，很可能有多个线程同时在同时运行它。比如你创建了一个`Semaphore(0)`，让N个线程执行`acquire()`，自然这多个线程都会阻塞在`acquire()`这里，然后你让另一个线程执行`release(N)`。

- 此时 释放共享锁的线程，肯定在执行doReleaseShared。
- 由于 上面这个线程的unparkSuccessor，head后继的代表线程也会唤醒，进而执行doReleaseShared。
- 重复第二步，获取共享锁的线程 又会唤醒 新head后继的代表线程。

观察上面过程，有的线程 因为CAS操作失败，或head变化（主要是因为这个），会一直退不出循环。进而，可能会有多个线程都在运行该函数。doReleaseShared源码分析]中的图解举例了一种循环继续的例子，当然，循环继续的情况有很多。

# 总结

- 共享锁与独占锁的最大不同，是共享锁可以同时被多个线程持有，虽然AQS里面没有成员用来保存持有共享锁的线程们。
- 由于共享锁在获取锁和释放锁时，都需要唤醒head后继，所以将其逻辑抽取成一个`doReleaseShared`的逻辑了。