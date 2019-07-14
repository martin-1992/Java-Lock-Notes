## 共享式获取锁和释放锁
　　共享式锁与独占式锁的区别在于同一时刻是否有多个线程同时获取到同步状态，在独占锁 [acquire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquire.md) 分析过，当前节点会先判断前个节点是否为头节点，再尝试获取同步状态，其他节点的前节点不为头节点，则不会尝试获取同步状态，所以同一时刻只有当前节点的前节点为头节点，才会尝试获取同步状态。<br />
　　独占式锁保证只有一个线程能对进行读写操作，类似 synchronized 的互斥锁。但是在读多写少的场景下，独占式锁效率慢，而共享式锁保证多线程进行读操作。

- tryAcquireShared，尝试获取同步状态，大于 0 表示获取成功；
- addWaiter，调用该方法，将线程包装节点，模式为共享式，添加到同步队列末尾；
- 获取该节点的前节点，进行判断是否为头节点。如果是，尝试获取同步状态；
- 同步状态获取成功，则设置当前节点为新的头节点，删除旧的头节点。

```java
public final void acquireShared(int arg) {
    // 尝试获取同步状态，大于 0 表示获取成功
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    // 将线程包装节点，模式为共享式，添加到同步队列末尾
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 获取该节点的前节点
            final Node p = node.predecessor();
            // 如果前节点为头节点，尝试获取同步状态，
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 设置当前节点为新的头节点
                    setHeadAndPropagate(node, r);
                    // 删除旧的头节点
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            // 唤醒下个节点
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```
