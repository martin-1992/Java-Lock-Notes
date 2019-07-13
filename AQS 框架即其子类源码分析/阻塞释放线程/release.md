
## release
　　当前线程获取同步状态，并完成逻辑处理后，就需要释放同步状态。release 方法包含两个步骤：

- tryRelease，由子类继承实现，释放独占锁的同步状态；
- unparkSuccessor，释放成功后，会调用该方法唤醒下一个节点。然后下个节点从阻塞状态唤醒，尝试调用 acquire 方法获取独占锁；

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        // 如果头部节点不为空，且锁状态不为 0
        if (h != null && h.waitStatus != 0)
            // 唤醒下一个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

### unparkSuccessor
　　唤醒下个节点，这里存在下个节点为空，或超时 / 中断。这时需要跳过该节点，从队尾 tail 获取一个节点（线程），并唤醒。之所以要从队尾获取，是因为存在下下个节点仍然为空，或超时 / 中断，所以从后（队尾）往前回溯获取。

```java
private void unparkSuccessor(Node node) {
    // 获取该节点的同步状态
    int ws = node.waitStatus;
    // 如果状态为 0，则设置为 0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 当前节点的下个节点
    Node s = node.next;
    // 如果下个节点为空，或者是大于 0，即超时或中断，这时从队尾 tail 从后往前找到
    // 一个节点不为超时或中断的，唤醒该节点（线程）
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 唤醒节点的线程
        LockSupport.unpark(s.thread);
}
```
