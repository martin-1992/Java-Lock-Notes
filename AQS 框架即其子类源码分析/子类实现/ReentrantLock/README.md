## ReentrantLock
　　ReentrantLock 为重入锁，即该锁能支持一个线程对资源的重复加锁。ReentrantLock 分为公平式和非公平式，默认是使用非公平锁。<br />
　　synchronized 支持隐式的重入锁，而 ReentrantLock 在调用 lock() 进行上锁时，已经获取锁的线程，能够再次调用 lock() 方法获取而不被阻塞。

```java
public ReentrantLock() {
    // 默认非公平锁
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### NonfairSync
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

#### acquire
　　acquire 方法采用模板设计模式，为 AQS 框架提供的模板方法，只需实现 tryAcquire 方法即可，其他步骤流程已经在 AQS 中定义好的：
  
- tryAcquire，在非公平锁的设计中，这里会再次使用 CAS 来尝试获取独占锁，无需判断 AQS 队列中是否有节点（线程）在等待，获取成功，则结束步骤流程，这是非公平锁来源的第二点；
- 如 tryAcquire 获取独占锁失败，则先调用 addWaiter 将线程包装成一个节点，加入到队列中；
- 调用 AQS 框架的 acquireQueued 方法，这里会再次使用 tryAcquire 方法，但前提是前一个节点（线程）为头部节点，跟公平锁流程一样；
-  根据 acquireQueued 的返回值判断在获取锁的过程中是否被中断，如果中断, 则调用 selfInterrupt 方法，使当前线程再次中断。

  
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

#### Sync 
　　继承 AQS 框架，**实现 tryAcquire 方法**，非公平模式获取独占锁。

```java
private final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;

    abstract void lock();

    /**
     * Performs non-fair tryLock.  tryAcquire is implemented in
     * subclasses, but both need nonfair try for trylock method.
     */
    final boolean nonfairTryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 调用 AQS 的方法，获取锁状态，0 为锁已释放，可获取。大于 0，则锁已经被获取
        int c = getState();
        if (c == 0) {
            // CAS 尝试索取锁，并改变 state 的状态
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
     * 释放锁
     */
    protected final boolean tryRelease(int releases) {
        // 获取 state 状态，释放锁，则减去锁计数
        int c = getState() - releases;
        // 判断当前线程是否是获取锁的线程
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
    
    /**
     * 判断当前线程是否获得锁
     */
    protected final boolean isHeldExclusively() {
        return getExclusiveOwnerThread() == Thread.currentThread();
    }
}
```

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

### lockInterruptibly
　　独占式获取锁，当遇到线程中断直接抛出异常, 获取失败。这里是调用 AQS 框架的 acquireInterruptibly 方法，即 AQS 使用模板设计方法定义好流程，其中需要交由子类 ReentrantLock 实现的，tryAcquire 可看前面。

```java
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

### tryAcquireNanos
　　独占式获取锁，当遇到线程中断或超时直接抛出异常, 获取失败，同样是为模板方法，调用 AQS 框架的 tryAcquireNanos 方法。
  
```java
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

### unlock
　　释放锁，调用 AQS 框架的 release 方法。

```java
public void unlock(){
    sync.release(1);
}
```

### 公平锁与非公平锁的区别：
　　首先，大部分类如 ReentrantLock、Semaphore、CountDownLatc 继承了框架 AbstractQueuedSynchronizer（AQS），即队列同步器。在这个框架中，会实现一个先进先出的队列。线程先去获取锁，如获取不到锁，则进入到队列中，这里公平锁与非公平锁流程都一样。<br />
　　这时已经有一个队列，队列里每个节点都为一个等待线程。对新线程的操作则体现出公平锁与非公平锁不同的地方：
  
- 非公平锁的情况下，新线程使用 CAS 抢锁，如果锁已经释放，这时可直接获得锁，这对那些已经在队列中排队的线程就显得不公平；
- 进入到队列后，可再次使用 CAS 尝试获取锁，而不用判断前面是否有节点等待，而公平锁会先判断前面是否有节点（线程）在等待；
- 如果非公平锁这两次 CAS 都不成功，则跟公平锁一样，放入到队列中排队等待。

　　非公平锁相比公平锁，其优势在于吞吐量比较大，性能更好。缺点是如果每次新线程都能获取到锁，则在队列中的线程会一直处于等待状态。
