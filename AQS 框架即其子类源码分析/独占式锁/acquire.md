
## 独占式
　　独占式，同一时刻仅有一个线程去获取同步状态（锁）。<br />
　　独占式锁获取，acquire(int arg)方法为 AQS 提供的模板方法，该方法为独占式获取锁，但是该方法对中断不敏感，**即对线程进行中断操作后，该线程会依然在 CLH 同步队列中等待获取同步状态（锁）。**
  
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

　　各方法定义如下：
  
- tryAcquire：由子类实现，尝试获取锁，获取成功则设置锁状态并返回 true，否则返回 false；
- addWaiter：如果 tryAcquire 返回 FALSE（获取锁失败），则调用该方法将当前线程加入到 CLH 同步队列尾部；
- acquireQueued：当前线程会根据公平性原则来进行阻塞等待（自旋循环），直到获取锁为止。返回当前线程在等待过程中是否中断过；
- selfInterrupt：产生一个中断。

#### tryAcquire
　　由子类实现，尝试获取锁，获取成功则设置锁状态并返回 true，否则返回 false。
  
```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

#### acquireQueued
　　acquireQueued 方法为一个自旋过程，即当前线程（Node）通过 addWaiter 方法后进入同步队列后，会进入一个自旋过程（for 循环）。当同步状态成功获取到锁后，并且当前线程的前一个为头部节点，则从自旋过程中退出，并返回当前线程在等待过程中有没有中断过。<br />
　　当前线程会一直尝试获取同步状态，当然前提是只有其前节点为头结点才能够尝试获取同步状态，理由：

- 保持 FIFO 同步队列原则；
- 头节点释放同步状态后，将会唤醒其后继节点，后继节点被唤醒后需要检查自己是否为头节点。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 中断标志
        boolean interrupted = false;
        // 自旋过程，为死循环
        for (;;) {
            // 获取前一个节点
            final Node p = node.predecessor();
            // 判断前节点是否为头节点，如果为头节点，存在两种情况，一种为头节点使用锁，另一种为头节点已经释放锁，为空
            // 节点，这时再次调用 tryAcquire 尝试获取锁
            if (p == head && tryAcquire(arg)) {
                // 当前节点获取锁成功，将当前节点设置为头部节点
                setHead(node);
                // 删除前节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取失败，中断标志为 true，当前线程被阻塞，等待前节点获取锁，唤醒当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        // 获取失败，则清除该节点
        if (failed)
            cancelAcquire(node);
    }
}
```

### selfInterrupt
　　当前线程产生一个中断。

```java
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```
