
## AbstractQueuedSynchronizer
　　AbstractQueuedSynchronizer，AQS 即队列同步器。它是一个框架，使用方法是继承，即子类通过继承同步器并实现它的抽象方法来管理同步状态，使用到 AQS 的子类有 ReentrantLock、Semaphore、CountDownLatch 等。 <br />
　　可将 AQS 看做一个模板方法模式，部分方法交由子类实现，其他如获取同步状态、FIFO 同步队列等细节则由 AQS 框架实现。交由子类实现的方法，**非公平的独占锁和公平的独占锁的实现方法，则在子类实现的 tryAcquire 方法中：**
  
```java
// 独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态，同步状态即为锁
protected boolean tryAcquire(int arg) {  
    throw new UnsupportedOperationException();  
}  

// 独占式释放同步状态
protected boolean tryRelease(int arg) {  
    throw new UnsupportedOperationException();  
}  

// 共享式获取同步状态，返回值大于等于0则表示获取成功，否则获取失败；
protected int tryAcquireShared(int arg) {  
    throw new UnsupportedOperationException();  
  
}  

// 共享式释放同步状态；
protected boolean tryReleaseShared(int arg) {  
    throw new UnsupportedOperationException();  
```  

　　在基于 AQS 构建的同步器中，只能在一个时刻发生阻塞，从而降低上下文切换的开销，提高吞吐量。同时在设计 AQS 时充分考虑了可伸缩性，因此 J.U.C 中所有基于 AQS 构建的同步器均可以获得这个优势。同时 CAS 在获取同步状态时实现多种方式：
  
- 独占锁和共享锁机制；
- 线程阻塞后，如需要取消，支持中断；
- 线程阻塞后，如有超时要求，支持超时后中断。

　　AbstractQueuedSynchronizer 会把所有的请求线程构成一个 CLH 队列， 每个线程包装成一个节点，只有一个时刻发生阻塞，从而降低上下文切换的开销。当一个（节点）线程执行完毕（lock.unlock()）时会激活自己的后续节点（线程）。<br />
　　注意，只有获取不到同步状态（锁）的线程，才会进入被保证成节点进入阻塞状态，而正在执行的线程不再队列中。一个同步器至少需要包含两个功能：

- 获取锁。如果允许，则获取锁，如果不允许就阻塞线程，直到同步状态允许获取；
- 释放锁。修改同步状态，并且唤醒等待线程。

### state
　　AQS 使用一个 int 类型的成员变量 state 来表示同步状态，即用于同步线程之间的共享状态，通过 CAS 和 volatile 保证其原子性和可见性。
  
- state > 0 时，已经获取了锁；
- state = 0 时，释放了锁。

　　它提供了三个方法（getState()、setState(int newState)、compareAndSetState(int expect,int update)）来对同步状态 state 进行操作，当然 AQS 可以确保对 state 的操作是安全的，即保证方法的原子性。

```java
/**
 * 同步状态
 */
private volatile int state;

/**
 * 返回当前的同步状态值
 */
protected final int getState() {
    return state;
}

/**
 * 设置新的同步状态值
 */
protected final void setState(int newState) {
    state = newState;
}

/**
 * 使用 CAS 来尝试获取锁，如同步状态 state 为期望值 expect，则将其更新为 udpate 值，
 * 比如，compareAndSetState(0, 1) 为尝试获取锁，并将同步状态从 0 更新为 1，前面提
 * 到同步状态为 0 时，表示该锁已释放。
 */
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

### CLH 同步队列
　　CLH 同步队列是一个 FIFO 双向队列，AQS 依赖它来完成同步状态的管理。当前线程如果获取同步状态失败时，AQS 会将当前线程以及等待状态等信息构造成一个节点（Node），并将其加入到 CLH 同步队列，同时会阻塞当前线程。当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态（锁）。<br />
　　在 CLH 同步队列中，一个节点表示一个线程，它保存着线程的引用（thread）、状态（waitStatus）、前节点（prev）、后节点（next）。

### Node
　　把线程包装为 Node 对象的主要原因，除了用 Node 构造供虚拟队列外，还用 Node 包装了各种线程状态：
  
-  CANCELLED = 1，由于超时或者中断，该节点（线程）会被取消，被取消的节点不会参与到锁竞争中，一直保持取消状态不会转变为其他状态；
- SIGNAL = -1， 该节点（线程）的后面节点处于等待状态，如果当前节点（线程）release 或 cancel ，将会通知后继节点（unpark），使后继节点的线程得以运行；
- CONDITION = -2，该节点（线程）处于条件队列（另一个队列中），当其他线程对 Condition 调用了 signal() 后，该节点会移到同步队列中，即加入到获取锁的队列中；
- PROPAGATE = -3，表示下一次共享式同步状态获取将会无条件地传播下去，即传播共享锁。

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

    /** 等待状态，为 0 */
    volatile int waitStatus;

    /** 前节点 */
    volatile Node prev;

    /** 后节点 */
    volatile Node next;

    /** 获取同步状态的线程 */
    volatile Thread thread;

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

### addWaiter
　　负责把当前无法获得锁的线程包装为一个 Node 添加到队尾。即创建一个新节点，并添加到 CLH 队列尾部。参数 mode 有两种模式，分别为独占锁模式和共享锁模式，默认为独占锁模式。将线程添加到队尾的流程：
  
- 如果当前队尾已经存在（tail != null），则使用 CAS 把当前线程更新为 tail；
- 如果当前 tail 为 nul，或线程调用 CAS 设置队尾失败，则通过 enq 方法继续设置 tail。enq 方法为自旋过程，直到将线程添加到队尾才推出。
  
```java
/**
 * 将当前线程包装成节点，并添加到队尾。
 * 
 * @param mode Node.EXCLUSIVE 为独占模式, Node.SHARED 为共享模式
 * @return 新节点
 */
private Node addWaiter(Node mode) {
    // 根据当前线程和节点模式（独占或共享）创建新节点
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

### enq
　　该方法就是循环调用 CAS，将节点（线程）添加到队列末尾。因为线程并发对 Tail 调用 CAS 可能会导致其他线程 CAS 失败，解决方法是循环 CAS 直到成功把当前线程追加到队尾（或设置队头）。总而言之，addWaiter 的目的就是通过 CAS 把当前线程追加到队尾，并返回包装后的 Node 实例。<br />

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
