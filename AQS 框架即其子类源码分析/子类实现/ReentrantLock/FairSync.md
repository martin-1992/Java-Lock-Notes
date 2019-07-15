### AbstractQueuedSynchronizer#acquire
　　公平锁的 acquire，为模板方法模式。在 tryAcquire 方法中，公平锁会使用 hasQueuedPredecessors 先判断同步队列中是否有线程要获取同步状态，是则退出。于是会调用 acquire 方法的 addWaiter、acquireQueued 将当前线程包装为节点添加到同步队列末尾。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
 ```

### FairSync#tryAcquire
　　对比非公平锁 [nonfairTryAcquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantLock/NonfairSync.md) 的代码，发现唯一不同的地方，是公平锁的 if 条件多了 hasQueuedPredecessors，它会先判断头节点的下个节点是否为空或者当前线程是否为同步队列中的节点，是则退出调用 acquire 方法的 addWaiter、acquireQueued 将当前线程包装为节点添加到同步队列末尾。<br />
　　而非公平锁没有这个 hasQueuedPredecessors 判断，于是先调用 CAS 先获取同步状态，获取失败才调用 acquire 方法的 addWaiter、acquireQueued 将当前线程包装为节点添加到同步队列末尾。<br />
　　总结公平锁跟非公平锁有两点区别，非公平锁会尝试调用两次 CAS 来获取同步状态：
  
- 在 lock() 方法中，非公平锁会先让当前线程使用 CAS 来获取锁，然后在调用 AQS 框架中的 acquire 方法，而公平锁则直接调用 acquire 方法；
- 在 tryAcquire() 方法中，公平锁会先判断 AQS 框架的同步队列中是否有线程在等待，如有，则当前线程加入队列中等待，否则调用 CAS 尝试获取同步状态。非公平锁则直接调用 CAS 来尝试获取同步状态。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 判断 AQS 框架的队列中是有线程在等待，如没有，则调用 CAS 尝试获取锁
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
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
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    // 判断头节点的下个节点是否为空；
    // 当前线程是否为同步队列中的节点
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
