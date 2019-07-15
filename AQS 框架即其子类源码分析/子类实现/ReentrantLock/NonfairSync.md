## NonfairSync
　　非公平锁，这里新线程使用 CAS 来获取锁，如果锁已经释放，这时可直接获得锁，而这对那些已经在队列中排队的线程就显得不公平，这是非公平锁来源的第一点。
  
```java
/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        // 新线程使用 CAS 来获取锁并改变 state 状态，当锁为 0 时，则更新为 1，即获取到锁
        if (compareAndSetState(0, 1))
            // 获取成功，则设置当前线程获得独占锁
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 获取不成功，则调用 CAS 框架的 acquire 方法来获取
            acquire(1);
    }
    
    /**
     * 实现 AQS 框架 的 acquire 方法中的 tryAcquire 方法，因 tryAcquire 为 protected，所以必须使用继承来实现。
     * tryAcquire 为获取独占锁方法，获取独占锁成功，则返回 True。
     */
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```
