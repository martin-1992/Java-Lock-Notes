
## release
　　当线程获取锁后，执行完相应逻辑后就需要释放锁，AQS 提供了 release(int arg) 方法释放锁。该方法同样是先调用由子类实现的 tryRelease(int arg) 方法来释放锁，释放成功后，会调用 unparkSuccessor(Node node) 方法唤醒下一个节点。
  
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
        // 唤醒下个节点
        LockSupport.unpark(s.thread);
}
```
