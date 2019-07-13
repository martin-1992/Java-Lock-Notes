
## ReadLock
　　读锁使用共享式，支持多线程获取，重入等。

```java
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;

    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    /**
     * 读锁使用共享式
     */
    public void lock() {
        sync.acquireShared(1);
    }

    /**
     * 读锁使用共享式中断锁
     */
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * 尝试获取读锁
     */
    public boolean tryLock() {
        return sync.tryReadLock();
    }

    /**
     * 读锁使用共享式中断超时锁
     */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * 读锁释放
     */
    public void unlock() {
        sync.releaseShared(1);
    }

    /**
     * 子类实现
     */
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }

    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
```       

### tryAcquireShared
　　使用 CAS 尝试获取读锁，获取成功后：
  
- 该线程为第一个获取读锁，或者是之前的线程释放了读锁（读锁数量减到 0），使用 firstReader 记录当前线程为第一个获取读锁的线程，读锁数量 + 1；
- 当前线程重复获取读锁，读锁数量 + 1；
- 使用 cachedHoldCounter，将其设置为当前线程，读锁数量 + 1。


```java
protected final int tryAcquireShared(int unused) {
    
    // 当前线程
    Thread current = Thread.currentThread();
    // getState() 为 AQS 框架的方法，state 大于 0，即有线程持有锁，等于 0，则锁已释放
    int c = getState();
    // 不等于 0，即有线程持有写锁，并且不是当前线程持有写锁，所以获取读锁失败，返回 -1。
    // 如果当前线程持有写锁，可继续往下走，获取读锁
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
        
    // 读锁的获取次数
    int r = sharedCount(c);
    // 当读锁不被阻塞，且读锁的获取次数没有溢出（超过最大值，2^16 -1），
    // 使用 CAS 尝试获取读锁，获取读锁成功，state 的高 16 位加一
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            // 该线程为第一个获取读锁，或者是之前的线程释放了读锁（读锁数量减到 0），
            // 使用 firstReader 记录当前线程为第一个获取读锁的线程，读锁数量 + 1
            firstReader  = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            // 当前线程重复获取读锁，重入加 1
            firstReaderHoldCount++;
        } else {
            // cachedHoldCounter ，用于缓存最后一个获取读锁的线程。当 cachedHoldCounter 为空，或缓存的不
            // 是当前线程，则设置为当前线程
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                // 因为 cachedHoldCounter 缓存的是当前线程，而当前线程会调用 readHolds.get() 
                // 进行初始化 ThreadLocal，需要重新设置
                readHolds.set(rh);
            // 缓存的读锁数加 1
            rh.count++;
        }
        return 1;
    }
    // 如果
    return fullTryAcquireShared(current);
}
```

#### fullTryAcquireShared
　　当读锁被阻塞，或 CAS 尝试获取读锁失败（与另一个读锁或写锁竞争失败），则会进行 fullTryAcquireShared。跟非公平模式（有两次 CAS）类似，这里会尝试再次获取锁，如果获取失败，才会被包装成节点，进入阻塞队列。

```java
final int fullTryAcquireShared(Thread current) {

    HoldCounter rh = null;
    // 自旋
    for (;;) {
        int c = getState();
        // 不是当前线程持有写锁，获取读锁失败，进入阻塞队列
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            // 阻塞队列中有其他节点（线程）在等待获取读锁
            if (firstReader == current) {
                // firstReader 重入获取读锁
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    // cachedHoldCounter  为空，或者缓存的不是当前线程，则到 
                    // ThreadLocal 中获取当前线程的 HoldCounter
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        // 如果读锁数为 0，由于调用 readHolds.get() 进行初始化 ThreadLocal 导致的，则移除该线程，
                        // 因为不知道读锁数，所以获取读锁失败，进入阻塞队列
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        // 溢出，则抛出异常
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 使用 CAS 尝试获取读锁，获取读锁成功后的处理逻辑与 tryAcquireShared 类似
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
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

### releaseShared

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

#### tryReleaseShared
　　释放锁包含两个步骤：

- 将 hold count 减 1；
    1. 如果为 firstReader，即当前线程为第一个持有读锁的线程，则 firstReaderHoldCount 减一，减到 0，则将 firstReader 置为空；
    2. 如果为 cachedHoldCounter，即最后一个持有读锁的线程，对 cachedHoldCounter.count 减一，减到 0，则将 ThreadLocal 移除掉，目的是防止一直引用，用于 GC。

- 使用 CAS 将 state 高 16 位减 1，如果读锁和写锁都释放光了，则唤醒后继的获取写锁的线程。
　　

```java
protected final boolean tryReleaseShared(int unused) {
    // 当前线程
    Thread current = Thread.currentThread();
    // 当前线程为第一个持有读锁的线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            // 当前线程持有该锁的次数为 1，释放后为空
            firstReader = null;
        else
            // 当前线程多次持有该锁，则减一
            firstReaderHoldCount--;
    } else {
        // 获取最后一个持有读锁的线程
        HoldCounter rh = cachedHoldCounter;
        // 判断持有读锁的线程是否为空，或不是当前线程，则到 ThreadLocal 中获取
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 当前线程持有读锁数小于等于1，则释放锁后为 0，需将 ThreadLocal 移除掉，防止一直引用，导致内存泄漏
            readHolds.remove();
            // lock() 一次，但 unlock() 多次，抛出异常
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 读锁数减一
        --rh.count;
    }
    
    for (;;) {
        // 对 state 进行减一操作
        int c = getState();
        // 使用 CAS 将 state 高 16 位减 1
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // 如果 nextc 为 0，表示 state 全部 32 位都为 0，即读锁和写锁都为空
            // 返回 true，用于唤醒下个节点（线程）以获取写锁
            return nextc == 0;
    }
}

private IllegalMonitorStateException unmatchedUnlockException() {
    return new IllegalMonitorStateException(
        "attempt to unlock read lock, not locked by current thread");
}
```
