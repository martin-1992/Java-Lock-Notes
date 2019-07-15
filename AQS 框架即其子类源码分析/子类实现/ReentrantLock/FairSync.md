### FairSync
　　公平锁，跟非公平锁有两点区别：
  
- 在 lock() 方法中，非公屏锁会先让当前线程使用 CAS 来获取锁，然后在调用 AQS 框架中的 acquire 方法，而公平锁则直接调用；
- 在 tryAcquire() 方法中，公平锁会先判断 AQS 框架的队列中是有线程在等待，如有，则当前线程加入队列中等待，否则调用 CAS 尝试获取锁。非公平锁则直接调用 CAS 来尝试获取锁。

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
```
