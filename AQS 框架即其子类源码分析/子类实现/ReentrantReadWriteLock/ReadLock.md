## ReadLock
　　读锁使用共享式，支持多线程获取，在没有其他写线程访问（或者写状态为 0）时，读锁总会成功获取。

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

- 获取当前线程和同步状态；
- 如果持有写锁，判断是否是当前线程持有的，不是则获取读锁失败，包装成节点添加到同步队列中；
- 获取持有写锁数，这里 readerShouldBlock 分为公平是和非公平式；
    1. [FairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantReadWriteLock/FairAndNonfair.md)，公平式会先判断同步队列中是否有节点线程在等待，是则结束，执行 fullTryAcquireShared 方法；
    2. [NonfairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantReadWriteLock/FairAndNonfair.md) ，非公平式先判断头节点的下个节点是否为写锁，是则让该节点获取写锁，避免写锁等待。
- 判断新加后的写锁次数有没溢出，没溢出在尝试使用 CAS 获取读锁；
- 判断读锁数；
    1. 如果为 0，表示当前线程为第一个获取读锁，设置 firstReader 为当前线程，firstReaderHoldCount = 2；
    2. 如果读锁数不为 0，还是这个线程获取读锁，则 firstReaderHoldCount ++；
    3. 如果读锁数不为 0，上次获取读锁的线程与当前线程不是同一个线程，则设置 cachedHoldCounter 为当前线程；
    4. 获取读锁成功；
 - if 条件失败时，即 readerShouldBlock 为 true 或是读锁数溢出或是调用 CAS 失败，则调用 fullTryAcquireShared 方法。

```java
// 计算持有的写锁数
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
// 计算持有的读锁数
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

protected final int tryAcquireShared(int unused) {

    // 当前线程
    Thread current = Thread.currentThread();
    // getState() 为 AQS 框架的方法，获取同步状态
    int c = getState();
    // 不等于 0，即有线程持有写锁，并且不是当前线程持有写锁，所以获取读锁失败，返回 -1。
    // 如果当前线程持有写锁，可继续往下走，进行锁降级
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
        
    // 持有的读锁数
    int r = sharedCount(c);
    // readerShouldBlock() 分为公平式和非公平式，公平式是会先判断同步队列是否有线程在等待，非公平式会判
    // 断头节点的下个节点（线程）是否为写锁，是则先让该节点获取写锁，避免写锁等待，不是则先让当前线程获取
    // 读锁。然后继续判断新加后的写锁次数有没溢出，没溢出在尝试获取读锁
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
            // cachedHoldCounter ，用于缓存最后一个获取读锁的线程。当 cachedHoldCounter 为空，或
            // 线程 id（th.tid）不是当前线程的 id，则设置为当前线程
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
    return fullTryAcquireShared(current);
}
```

#### fullTryAcquireShared
　　fullTryAcquireShared 方法与 tryAcquireShared 中的代码有重复部分，但总体上更简单。

- 获取同步状态；
- 写锁数不为 0，且不是当前线程持有写锁，获取读锁失败，包装成节点进入同步队列；
- readerShouldBlock 分为公平式和非公平式，前面有讲到。如果同步队列中有节点（线程）在等待，则从 ThreadLocal 中获取当前线程的 HoldCounter，如果读锁数为 0，是由于调用 readHolds.get() 进行初始化 ThreadLocal 导致的，则移除该线程，因为不知道读锁数，所以获取读锁失败，进入同步队列；
- 判断读锁数，是否溢出，是则抛出异常；
- 使用 CAS 尝试获取读锁，获取读锁成功后的处理逻辑与 tryAcquireShared 类似。



```java
final int fullTryAcquireShared(Thread current) {
    
    HoldCounter rh = null;
    // 自旋
    for (;;) {
        // 获取同步状态
        int c = getState();
        // 写锁数不为 0，且不是当前线程持有写锁，获取读锁失败，包装成节点进入同步队列
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        } else if (readerShouldBlock()) {
            // 同步队列中有其他节点（线程）在等待获取读锁，判断当前线程是否为第一个获取读锁的线程
            if (firstReader == current) {
                // firstReader 重入获取读锁
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    // cachedHoldCounter 为空，或者缓存的不是当前线程，则到 
                    // ThreadLocal 中获取当前线程的 HoldCounter
                    if (rh == null || rh.tid != getThreadId(current)) {
                        // 获取当前线程的 HoldCounter
                        rh = readHolds.get();
                        // 如果读锁数为 0，由于调用 readHolds.get() 进行初始化 ThreadLocal 导致的，则移除该线程，
                        // 因为不知道读锁数，所以获取读锁失败，进入同步队列
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
　　释放读锁。

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
　　释放读锁包含两个步骤：

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
