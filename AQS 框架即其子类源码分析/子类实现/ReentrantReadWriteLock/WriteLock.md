
### WriteLock
　　写锁为独占锁，如果有读锁被占用，写锁获取是要进入到阻塞队列中等待的。

```java
public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    /**
     * 调用 AQS 框架的 acquire 方法，使用独占锁，该方法需要由子类实现 tryAcquire 方法
     */
    public void lock() {
        sync.acquire(1);
    }

    /**
     * 独占锁中断方法
     */
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    /**
     * 
     */
    public boolean tryLock( ) {
        return sync.tryWriteLock();
    }

    /**
     * 独占锁中断超时方法
     */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    /**
     * 释放锁
     */
    public void unlock() {
        sync.release(1);
    }

    /**
     * 
     */
    public Condition newCondition() {
        return sync.newCondition();
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

    /**
     * 
     */
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    /**
     * 获取写锁数量（包含重入次数）
     */
    public int getHoldCount() {
        return sync.getWriteHoldCount();
    }
}
```

### tryAcquire
　　写锁使用独占式，保证多线程下，只有一个线程能获得。
  
  
```java
protected final boolean tryAcquire(int acquires) {
    // 当前线程
    Thread current = Thread.currentThread();
    // 调用 AQS 方法，获取当前节点（线程）的状态
    int c = getState();
    // 节点低位为写锁数量，高位为读锁数量
    int w = exclusiveCount(c);
    if (c != 0) {
        // 写锁数量为 0，即没有线程持有写锁；c 不等于 0，表示有线程持有读锁或写锁，且
        // 不是当前线程持有的。只要有读锁或写锁被占用，则不能获取到写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 溢出抛出异常
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 更新重入数量，不需要使用 CAS，因为写锁只有一个线程能持有
        setState(c + acquires);
        return true;
    }
    // c 为 0，表示读锁和写锁都为 0，如果写锁不阻塞，且用 CAS 尝试获取写锁，
    // 获取成功，则设置当前线程为获取写锁的线程
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

### unlock()
　　释放写锁，释放成功后，唤醒下个节点。

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒下个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

#### tryRelease
　　写锁为独占锁，多线程下只有一个线程能持有，为线程安全的。只需将 state -1 即可。如果 state 减到 0，即所有写锁都释放了，会唤醒下个节点。

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获取锁状态
    int nextc = getState() - releases;
    // 写锁为 0，所有写锁都释放，包括重入的，将线程置为空
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    // 如返回 true，则会唤醒下个节点，以进行获取锁操作
    return free;
}
```
