# java_concurrent_locks_analysis

## [AbstractQueuedSynchronizer](https://github.com/martin-1992/Java-Lock-Notes/tree/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/AbstractQueuedSynchronizer)
　　AbstractQueuedSynchronizer，AQS 即队列同步器。它是一个框架，使用方法是继承，即子类通过继承同步器并实现它的抽象方法来管理同步状态，使用到 AQS 的子类有 ReentrantLock、Semaphore、CountDownLatch 等。 <br />
　　之所以将 AQS 称做框架，是因为它使用[模板设计模式](https://github.com/martin-1992/head_first_design_patterns_notebook/tree/master/chapter_8)，实现了大部分细节，包括 FIFO 队列、将线程包装成节点、阻塞线程等。有独占式锁和共享式锁，非公平锁和公平锁则由子类通过实现 tryAcquire 或 tryAcquireShared。比如 子类 ReentrantLock 实现 tryAcquire，包含公平独占锁和非公平独占锁。

### 独占式锁
- [acquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquire.md)，独占式获取锁。对中断不敏感，即对线程进行中断操作后，该线程会依然在 CLH 同步队列中等待获取同步状态（锁）；
- [acquireInterruptibly](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquireInterruptibly.md)，独占式获取响应中断。在 acquire 原有基础上，加上线程中断则抛出异常；
- [tryAcquireNanos](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/tryAcquireNanos.md)，独占式超时获取。在 acquireInterruptibly 的中断基础上，加入超时控制，即判断线程在指定时间是否获得锁。

### 共享式锁

### 阻塞释放线程
- [release](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E9%98%BB%E5%A1%9E%E9%87%8A%E6%94%BE%E7%BA%BF%E7%A8%8B/release.md)，释放锁，唤醒下一个节点（线程）尝试获取锁；
- [shouldParkAfterFailedAcquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E9%98%BB%E5%A1%9E%E9%87%8A%E6%94%BE%E7%BA%BF%E7%A8%8B/shouldParkAfterFailedAcquire.md)，当前节点（线程）获取锁失败后，给前节点打上 SIGNAL 标识，这样当前线程会被阻塞。直到前节点在释放锁后，会根据这个标识来决定是否唤醒当前节点（线程）来获得锁。

### 子类实现
- [ReentrantLock](https://github.com/martin-1992/java_concurrent_locks_analysis/blob/master/%E5%AD%90%E7%B1%BB%E5%AE%9E%E7%8E%B0/ReentrantLock.md)，继承 AQS 框架，实现 tryAcquire 方法，包含公平模式和非公平模式获取独占锁；

### 公平锁与非公平锁的区别：
　　首先，大部分类如 ReentrantLock、Semaphore、CountDownLatc 继承了框架 AbstractQueuedSynchronizer（AQS），即队列同步器。在这个框架中，会实现一个先进先出的队列。线程先去获取锁，如获取不到锁，则进入到队列中，这里公平锁与非公平锁流程都一样。<br />
　　这时已经有一个队列，队列里每个节点都为一个等待线程。对新线程的操作则体现出公平锁与非公平锁不同的地方：

- 非公平锁的情况下，新线程使用 CAS 抢锁，如果锁已经释放，这时可直接获得锁，这对那些已经在队列中排队的线程就显得不公平；
- 进入到队列后，可再次使用 CAS 尝试获取锁，而不用判断前面是否有节点等待，而公平锁会先判断前面是否有节点（线程）在等待；
- 如果非公平锁这两次 CAS 都不成功，则跟公平锁一样，放入到队列中排队等待。

　　非公平锁相比公平锁，其优势在于吞吐量比较大，性能更好。缺点是如果每次新线程都能获取到锁，则在队列中的线程会一直处于等待状态。
  
### reference:

- https://mp.weixin.qq.com/s/-swOI_4_cxP5BBSD9wd0lA 
- https://www.jianshu.com/p/279baac48960 
- https://blog.csdn.net/vernonzheng/article/details/8275624
- https://www.jianshu.com/p/3f3417dbcac4 
