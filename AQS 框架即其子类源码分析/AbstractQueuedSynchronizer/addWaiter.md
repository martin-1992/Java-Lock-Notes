### addWaiter
　　获取锁失败后，执行该方法，**将当前无法获得锁的线程包装为一个节点 Node，使用 compareAndSetTail 添加到同步队列尾部，** expect 为传入当前线程认为的尾节点，update 为要添加的节点。<br />
　　参数 mode 有两种模式，分别为独占锁模式和共享锁模式，默认为独占锁模式。将线程添加到队尾的流程：

- 根据当前线程和节点模式（独占或共享），包装为新节点；
- 如果当前队尾已经存在（pred != null），则使用 CAS 把当前节点设置为新的队尾 tail；
- 如果 pred 为 nul，或线程调用 CAS 设置队尾失败，则通过 enq 方法继续设置 tail。enq 方法为自旋过程，直到将线程添加到队尾才退出。

```java
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

private Node addWaiter(Node mode) {
    // 根据当前线程和节点模式（独占或共享），包装为新节点
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试快速添加尾节点，失败则使用 enq 方法
    Node pred = tail;
    // 队列中有节点，新节点添加到末尾
    if (pred != null) {
        // 将 tail 指向新节点，新节点的 prev 为旧节点
        node.prev = pred;
        // 使用 CAS 设置尾节点
        if (compareAndSetTail(pred, node)) {
            // 旧节点的 next 指向新节点
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

#### enq
　　该方法就是循环调用 CAS，将节点（线程）添加到队列末尾，只有添加成功，才能从该方法返回。<br />
　　因为线程并发对 Tail 调用 CAS 可能会导致其他线程 CAS 失败，解决方法是循环 CAS 直到成功把当前线程追加到队尾（或设置队头）。

```java
/**
 * 将节点插入队列，必要时进行初始化
 *
 * @param node 要插入的新节点
 * @return 新节点的前一个节点
 */
private Node enq(final Node node) {
    // 循环处理，直到能插入队列返回值为止
    for (;;) {
        // 当尾部节点 tail 为空时，即为空队列，将 tail 设为 head，
        // 这时在循环处理，判断不为空，于是进行添加新节点操作
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 新节点的 prev 指向旧节点 t，旧节点的 next 指向新节点，添加成功返回旧节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
