
## AbstractQueuedSynchronizer
　　AbstractQueuedSynchronizer，AQS 即队列同步器。它是一个框架，用来构建锁或其他同步组件的基础框架，使用方法是继承，即子类通过继承同步器并实现它的抽象方法来管理同步状态，通过内置的先进先出（FIFO）队列来完成资源获取线程的排队工作，使用到 AQS 的子类有 [ReentrantLock]()、[Semaphore]()、[CountDownLatch]() 等。 <br />
　　同步器提供的三个方法来获取和修改同步状态，getState()、setState(int newState) 和 compareAndSetState(int expect, int update) 这三个方法能够保证状态的改变是安全的。<br />
　　AQS 框架用到了模板方法模式，部分方法交由子类实现，AQS 框架简化锁的实现方式，负责管理同步状态、线程的排队、等待与唤醒线程，子类继承 AQS，只需实现模板方法中的同步方法。比如 [acuqire](https://github.com/martin-1992/Java-Lock-Notes/blob/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/%E7%8B%AC%E5%8D%A0%E5%BC%8F%E9%94%81/acquire.md) 独占锁方法，子类只需实现 tryAcquire 方法，其他流程，如获取同步状态失败，将当前线程包装成节点，添加到队列中则由 acuqire 方法实现。即子类继承 AQS 框架重写指定的方法（如 tryAcquire），在调用同步器的模板方法（如 acuqire），则会调用子类重写的方法（tryAcquire）。<br />
 　　AQS 框架提供的模板方法分为三类：独占式锁的获取和释放、共享式锁的获取和释放、查询同步队列中的等待线程。

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

### 同步状态 state
　　AQS 使用一个 int 类型的成员变量 state 来表示同步状态，即用于同步线程之间的共享状态，通过 CAS 和 volatile 保证其原子性和可见性。
  
- state > 0 时，已经获取了锁；
- state = 0 时，释放了锁。

　　对同步状态 state 进行操作的方法如下，这些方法可确保对状态的改变是线程安全的。

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

### 同步队列
　　同步队列是一个 FIFO 双向队列，AQS 通过它来完成同步状态的管理。当前线程获取同步状态失败时，AQS 会将当前线程以及等待状态等信息构造成一个节点（Node），并将其加入到同步队列，同时会阻塞当前线程。当同步状态释放时，会把首节点中的线程唤醒（公平锁），使其再次尝试获取同步状态（锁）。<br />
　　在同步队列中，一个节点表示一个线程，节点保存着获取同步状态（锁）失败的线程引用（thread）、等待状态（waitStatus）、前节点（prev）、后节点（next）。

#### 节点 Node
　　把线程包装为 Node 对象的主要原因，除了用 Node 构造供虚拟队列外，还用 Node 包装了各种线程状态 waitStatus：
  
- CANCELLED = 1，由于在同步队列中等待的线程等待超时或者被（其它线程）中断，该节点（线程）会从同步队列中取消等待，被取消的节点不会参与到锁竞争中，一直保持取消状态不会转变为其他状态；
- SIGNAL = -1， 该节点（线程）的后面节点处于等待状态，如果当前节点（线程）释放了同步状态或被取消，将会通知后继节点（unpark），使后继节点的线程得以运行；
- CONDITION = -2，该节点（线程）处于条件队列（另一个队列中），当其他线程对 Condition 调用了 signal() 后，该节点会移到同步队列中，即加入到获取锁的队列中；
- PROPAGATE = -3，表示下一次共享式同步状态获取将会无条件地传播下去，即传播共享锁；
- INITIAL = 0，初始状态。

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

　　通过节点构成同步队列，如下图，同步队列是拥有头节点和尾节点的链表。

![avatar](photo_1.png)

### addWaiter
　　负责把当前无法获得锁的线程包装为一个节点 Node 添加到同步队列尾部，将节点接入到队尾，需保证线程安全，所以使用了 compareAndSetTail(Node expect, Node update)，用法同 CAS 一样，传入当前线程认为的尾节点和要添加的节点。如果同步队列的尾节点是当前线程认为的尾节点，即没有其它线程操作过，则进行添加。如果同步队列的尾节点不是当前线程认为的尾节点，则表示有其他线程已添加新的尾节点，需重新从同步队列中获取新的尾节点，在重复前面的流程。
　　参数 mode 有两种模式，分别为独占锁模式和共享锁模式，默认为独占锁模式。将线程添加到队尾的流程：
  
- 如果当前队尾已经存在（tail != null），则使用 CAS 把当前线程更新为 tail；
- 如果当前 tail 为 nul，或线程调用 CAS 设置队尾失败，则通过 enq 方法继续设置 tail。enq 方法为自旋过程，直到将线程添加到队尾才退出。
  
```java
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

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
