## NonfairSync
　　非公平锁，这里新线程使用 CAS 来获取同步状态（锁），如果同步状态已经释放（CAS 的 expect 为 0），这时可直接获得锁，而这对那些已经在同步队列中排队的线程就显得不公平，这是非公平锁来源的第一点。
  
```java
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

### acquire
　　[acquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquire.md) 方法采用模板设计模式，为 AQS 框架提供的模板方法，只需实现 tryAcquire 方法即可，其他步骤流程已经在 AQS 中定义好的：

- tryAcquire，在非公平锁的设计中，这里会再次使用 CAS 来尝试获取独占锁，无需判断 AQS 队列中是否有节点（线程）在等待，获取成功，则结束步骤流程，这是非公平锁来源的第二点；
- addWaiter，如果 tryAcquire 获取同步状态失败，返回 false，则调用该方法构造同步节点（模式为独占式 Node.EXCLUSIVE），并使用 CAS 将该节点加入到同步队列的尾部；
- 调用 AQS 框架的 acquireQueued 方法，这里会再次使用 tryAcquire 方法，但前提是前一个节点（线程）为头部节点，跟公平锁流程一样；
- 根据 acquireQueued 的返回值判断在获取锁的过程中是否被中断，如果中断, 则调用 selfInterrupt 方法，使当前线程再次中断。
  
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

### nonfairTryAcquire 
　　继承 AQS 框架，**实现 tryAcquire 方法**，非公平模式获取独占锁。

- 获取当前线程；
- 获取同步状态，如果同步状态为 0，表示没有线程获得锁，则调用 CAS 尝试获取同步状态；
- 如果同步状态不为 0，表示有线程获得锁。判断锁的线程是不是当前线程，是则将当前锁状态 c 加上 acquires，为新的锁状态（计数值）；

```java
private final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    final boolean nonfairTryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 调用 AQS 的方法，获取同步状态，0 为锁已释放，可获取。大于 0，则同步已经被获取
        int c = getState();
        if (c == 0) {
            // CAS 尝试索取同步状态，并改变同步状态 acquires
            if (compareAndSetState(0, acquires)) {
                // 成功获取锁，并设置当前线程为获取独占锁的线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            // 如果当前线程是获取锁的线程，state 状态加一（重入获取锁计数）
            // 并调用 setState 设置状态
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
    
   /**
     * 判断当前线程是否获得锁
     */
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }
```

#### tryRelease
　　释放锁，即减去同步状态的值。

- 当前同步状态的值，减去要释放的锁次数；
- 先当前线程是不是获取锁的线程，不是抛出异常；
- 如果当前同步状态的值减去锁计数后，同步状态为 0，则表示锁已释放，将当前线程置为空，用于 GC 会后，并返回 true；
- 如果当前同步状态的值减去锁计数后，同步状态不为 0，则表示锁还持有在当前线程上，则返回 false。

```java
    /**
     * 释放锁
     */
    protected final boolean tryRelease(int releases) {
        // 获取 state 状态，释放锁，则减去锁计数
        int c = getState() - releases;
        // 当前线程不是获取锁的线程，则释放同步状态时抛出异常
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            // state 为 0，锁已释放，将当前线程置为空
            free = true;
            setExclusiveOwnerThread(null);
        }
        // 重新设置锁状态
        setState(c);
        return free;
    }
}
```
