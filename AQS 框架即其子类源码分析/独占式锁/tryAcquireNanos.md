
## tryAcquireNanos
　　tryAcquireNanos(int arg,long nanos)，独占式超时获取，该方法为 acquireInterruptibly 方法的进一步增强，除了响应中断外，还有超时控制。即如果当前线程没有在指定时间内获取同步状态，则会返回 false，否则返回 true。

### tryAcquireNanos
　　与 acquireInterruptibly 不同的地方在于多了个超时控制。首先校验该线程是否已经中断了，如果是则抛出 InterruptedException，否则执行 tryAcquire(int arg) 方法在指定时间获取同步状态，如果获取成功，则直接返回，否则执行 doAcquireInterruptibly(int arg)。

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 线程中断，则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 获取同步状态，如在指定时间没获取成功，则会执行 doAcquireNanos
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

### doAcquireInterruptibly

- 计算超时时间，超时则返回 false；
- 如没超时，但接近超时（小于 spinForTimeoutThreshold），则直接自旋超时返回 false，因为调用 LockSupport 需要开销；
- 没超时，且剩余时间大于 spinForTimeoutThreshold，则使用 LockSupport 使当前节点（线程）进入阻塞状态。
　　
  
```java
static final long spinForTimeoutThreshold = 1000L;

private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    // 将当前线程保证成节点，加入到队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        // 自旋
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 获取失败，计算剩余的时间
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                // 超时则返回 false
                return false;
            // 若没超时, 并且大于 spinForTimeoutThreshold, 则线程被阻塞，等待唤醒，这里没超时
            // 但小于spinForTimeoutThreshold, 是直接自旋, 因为效率更高，调用 LockSupport 是
            // 需要开销的
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                // 线程此时唤醒是通过线程中断，则直接抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
