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

　　实现重入锁需考虑两个方面：

- 锁需要判断再次获取锁的线程是否为之前已经获得锁的线程；
- 需要有个计数器计算，上锁 n 次，释放锁就减去 n 次 的 -1。当计数器减到 0 时，其他锁可以尝试获得锁。在 ReentrantLock 使用同步状态作为计数值，同步状态大于 0，即为获取锁，比如同步状态为 5，表示上锁 5 次。

### [NonfairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantLock/NonfairSync.md)
　　非公平锁，实现 acquire 方法的 tryAcquire。在 lock() 方法中会先调用 CAS 获取同步状态。获取不到在调用 AbstractQueuedSynchronizer#acquire 方法，然后调用非公平锁的 tryAcquire，再次调用 CAS 获取同步状态。获取失败，则调用 AbstractQueuedSynchronizer#acquire 方法的 addWaiter、acquireQueued 将当前线程包装为节点添加到同步队列末尾。

### [FairSync](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantLock/FairSync.md)
　　公平锁，实现 acquire 方法的 tryAcquire，与非公平锁唯一不同的地方在于多了个 hasQueuedPredecessors，即先判断头节点的下个节点是否为空或者当前线程是否为同步队列中的节点，是则退出调用 AbstractQueuedSynchronizer#acquire 方法的 addWaiter、acquireQueued 将当前线程包装为节点添加到同步队列末尾。

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
