### 节点 node
　　把线程包装为 Node 对象的主要原因，除了用 Node 构造供虚拟队列外，还用 Node 包装了各种线程状态 waitStatus：

- INITIAL = 0，节点初始时的默认状态；
- CANCELLED = 1，该节点（线程）获取锁的请求已取消，即不获取锁，可能是该线程等待超时或被中断；
- SIGNAL = -1， 该节点（线程）已准备好，等待锁释放。如果当前节点（线程）释放了同步状态或被取消，将会通知后继节点（unpark），使后继节点的线程得以运行；
- CONDITION = -2，该节点（线程）处于条件队列（另一个队列中），等待唤醒。当其他线程对 Condition 调用了 signal() 后，该节点会移到同步队列中，即加入到获取锁的队列中；
- PROPAGATE = -3，表示下一次共享式同步状态获取将会无条件地传播下去，即传播共享锁。只有在当前线程处于 SHARED 情况下，才能使用。

　　节点（线程）有两种模式等待获取锁，共享模式 SHARED 和独占模式 EXCLUSIVE。

```java
static final class Node {
    
    /** 节点为共享模式*/
    static final Node SHARED = new Node();
    
    /** 节点为独占模式 */
    static final Node EXCLUSIVE = null;

    static final int CANCELLED =  1;
    
    static final int SIGNAL    = -1;
    
    static final int CONDITION = -2;
    
    static final int PROPAGATE = -3;

    /** 当前节点在等待队列中的状态 */
    volatile int waitStatus;

    /** 前节点 */
    volatile Node prev;

    /** 后节点 */
    volatile Node next;

    /** 该节点的线程 */
    volatile Thread thread;
    
    /**
     * 等待队列中的后节点，如果当前节点是共享的，那么这个字段将是一个 SHARED 常量，也就是
     * 说节点类型（独占和共享）和等待队列中的后节点共用同一个字段
     */
    Node nextWaiter;

    /**
     * 判断是否为共享模式
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * 返回上一个节点，为空则抛出异常。
     */
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    /**
     * 建立初始头部节点或共享模式标记。
     */
    Node() {    
    }
    
    /**
     * 构造器方法
     */
    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

　　通过节点构成同步队列，如下图，同步队列是拥有头节点和尾节点的链表，遵循先进先出原则。<br />
　　头节点是获取同步状态成功的节点，头节点的线程在释放同步状态时，将会唤醒后续节点，而后续节点在获取同步状态成功时，会将自己设置为头节点。

![avatar](photo_1.png)