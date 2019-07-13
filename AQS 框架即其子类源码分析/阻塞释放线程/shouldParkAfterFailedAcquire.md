
## shouldParkAfterFailedAcquire
　　当前节点（线程）获取锁失败后，需要给前节点打上 SIGNAL 标识，这样当前线程会被阻塞。直到前节点在释放锁后，会根据这个标识来决定是否唤醒当前节点（线程）来获得锁。流程：
  
- 获取前节点的状态，非 SINNAL，非 CANCELLED 则使用 CAS 设为 SIGNAL 返回 false。于是在 acquireQueued 的自旋过程中，继续尝试获取锁；
- 如前节点的状态大于 0，即为 CANCELLED，此状态只有超时或被中断才有，于是删除该节点；
- 第一步已经将前节点设置为 SIGNAL，再次使用 shouldParkAfterFailedAcquire 会返回 true；
- 这时会调用 parkAndCheckInterrupt，当前线程被阻塞，等待前节点获取锁，唤醒当前线程。


```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前节点的状态
    int ws = pred.waitStatus;
    // 判断前节点的状态是否为 SIGNAL，是则当前线程处于等待状态
    // 如果前节点释放锁时，可唤醒当前线程
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        // 前节点状态为 CANCELLED，只有 CANCELLED 状态大于 0，
        // 表示该节点在获取过程中被中断或超时，删除该节点
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 这里状态为 0 或 PROPAGATE，使用 CAS 将前节点的状态标识为 SIGNAL，以便可唤醒后续节点（线程）                              
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

### parkAndCheckInterrupt
　　阻塞当前线程，同时返回当前线程的中断状态

```java
private final boolean parkAndCheckInterrupt() {
    // 阻塞当前线程
    LockSupport.park(this);
    // 会清除中断标识，并返回上次的中断标识
    return Thread.interrupted();
}
```

### cancelAcquire
　　清除因中断、超时而没有获取到锁的线程节点。

```java
/**
  * 清除因中断、超时而没有获取到锁的线程节点
  */
private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;
    
    // 该节点的线程引用为空
    node.thread = null;

    // 获取前节点
    Node pred = node.prev;
    // 如果前节点状态大于 0，即为 CANCELLED，因为它等于1
    // 于是也将前节点清除掉
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // 获取 pred.next，下面 CAS 操作需要用到
    Node predNext = pred.next;

    // 该节点设置为取消状态，需要清除掉
    node.waitStatus = Node.CANCELLED;

    // 如果清除的节点是尾节点，使用 CAS 设置 pred 为新的尾节点
    // 若需要清除额节点是尾节点, 则直接 CAS pred为尾节点
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            // 后继节点需要唤醒(但这里的后继节点predNext已经 CANCELLED 了)
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             // 将 pred 标识为 SIGNAL
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // 如果 pred 是头节点，则此刻可能有节点刚刚进入队列中 ,所以进行一下唤醒
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```
