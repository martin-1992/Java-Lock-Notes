
### WriteLock
　　写锁为独占锁，lock() 方法会调用 AQS 框架的 [acquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquire.md) 模板方法，其中抽象方法 [tryAcquire()](#tryAcquire)，由子类读写锁实现。<br />
　　获取写锁的条件：

- 同步状态为 0，表示没有线程持有写锁和读锁；
- 同步状态不为 0，当前线程持有写锁，可进行重入，增加写状态。

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

    public Condition newCondition() {
        return sync.newCondition();
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

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

### tryAcquire<a id='tryAcquire'></a>
　　写锁使用独占式，保证多线程下，只有一个线程能获得。

- 获取当前线程、同步状态；
- 如果同步状态不为 0，表示有线程持有读锁或写锁，再进行判断；
    1. w == 0，表示有线程持有读锁，这时无法获取写锁，返回 false。执行 AQS#acquire 方法的后续流程，将当前线程包装成节点，添加到同步队列末尾；
    2. 写锁数不为 0，有线程持有写锁，判断如果不是当前线程持有写锁，返回 false。同上执行 AQS#acquire 方法的后续流程；
    3. 先判断加上新的写锁数，是否会溢出，溢出则抛出异常；
    4. 当前线程持着写锁，则更新写锁计数，因为写锁只有一个线程持有，所以不用使用 CAS。
- 如果同步状态为 0，表示读锁和写锁的持有数都为 0。writerShouldBlock，为抽象方法。分别为公平锁 [ReentrantReadWriteLock#FairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantReadWriteLock/FairAndNonfair.md) 和非公平锁[ReentrantReadWriteLock#NonfairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantReadWriteLock/FairAndNonfair.md)，这时根据公平式和非公平式分两种情况；
    1. 公平式，会先判断同步队列是否有线程在等待，是则返回 false。执行 AQS#acquire 方法的后续流程；
    2. 非公平式，尝试执行 CAS 获取同步状态，获取成功，返回 true。
 
```java
// 根据同步状态计算写锁数量
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

// 为抽象方法，有两种实现方法，分别为公平锁 ReentrantReadWriteLock#FairSync 
// 和非公平锁 ReentrantReadWriteLock#NonfairSync
abstract boolean writerShouldBlock();

protected final boolean tryAcquire(int acquires) {
    // 当前线程
    Thread current = Thread.currentThread();
    // 调用 AQS 方法，获取同步状态
    int c = getState();
    // 根据同步状态计算写锁数量
    int w = exclusiveCount(c);
    if (c != 0) {
        // c 不等于 0，表示有线程持有读锁或写锁。然后 w == 0，表示写锁数量为 0，综合起来
        // 是有线程持有读锁，这时无法获取写锁，返回 false。第二种情况是写锁数量不为 0，但
        // 不是当前线程持有写锁，返回 false，不能获取写锁
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 溢出抛出异常
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 当前线程持着写锁，则更新写锁计数，因为写锁只有一个线程持有，所以不用使用 CAS
        setState(c + acquires);
        return true;
    }
    // 非公平锁这里 writerShouldBlock 会返回 false，然后调用 CAS 成功获取同步状态，则会
    // 设置当前线程持有同步状态，并返回 true
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 写锁不阻塞
    setExclusiveOwnerThread(current);
    return true;
}
```

### unlock()
　　释放写锁状态，当写锁数为 0 时，即所有写锁已释放，则唤醒下个节点。

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
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}

protected final boolean tryRelease(int releases) {
    // 不是当前线程持有写锁，抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获取释放后的写锁状态
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
