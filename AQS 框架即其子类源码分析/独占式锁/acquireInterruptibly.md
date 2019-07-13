
## acquireInterruptibly
　　AQS 提供了 acquireInterruptibly(int arg) 方法，即独占式获取响应中断。该方法在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断抛出异常 InterruptedException。

### acquireInterruptibly
　　首先校验该线程是否已经中断了，如果是则抛出 InterruptedException，否则执行 tryAcquire(int arg) 方法尝试获取锁，如果获取成功，则直接返回，否则执行 doAcquireInterruptibly(int arg) 将线程包装成节点，进入队列中。

```java
public final void acquireInterruptibly(int arg) throws InterruptedException {
    // 线程中断，则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 为 false，即锁获取不到时，执行 doAcquireInterruptibly
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

#### doAcquireInterruptibly
　　doAcquireInterruptibly(int arg) 方法与 acquire(int arg) 方法仅有两个差别：
  
- 方法声明抛出 InterruptedException 异常；
- 在中断方法处不再是使用 interrupted 标志，而是直接抛出 InterruptedException 异常。
　　
```java
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 直接抛出异常，不设置中断标志为 true
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
