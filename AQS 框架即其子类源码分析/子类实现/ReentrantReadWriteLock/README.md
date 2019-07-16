## ReentrantReadWriteLock
　　ReentrantReadWriteLock 为读写锁，同一时刻允许多个读线程访问，但在写线程访问时，所有的读线程和其他写线程均被阻塞。读写锁维护一对锁，一个读锁和一个写锁。<br />

### 读写状态的设计
　　AQS 框架的同步状态是一个整型变量，为了在一个整型变量上维护读锁和写锁，于是 “按位切割使用” 这个变量，如下图，将 32 位的整型变量切成两部分，高 16 位表示读锁，低 16 位表示写锁。

![avatar](photo_1.png)

```java
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 6317671515068378041L;

        // 写锁和读锁的常量
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        // 读锁，当前同步状态为 c，读状态为 c >>> 16
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        // 写锁，当前同步状态为 c，写状态为 c & ((1 << SHARED_SHIFT) - 1)，即 c & 0x0000FFFF
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
        // ...
}
```

