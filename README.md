## Java-Lock-Notes

### [AQS 框架即其子类源码分析](https://github.com/martin-1992/Java-Lock-Notes/tree/master/AQS%20%E6%A1%86%E6%9E%B6%E5%8D%B3%E5%85%B6%E5%AD%90%E7%B1%BB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)


### [Java CAS 原理](https://github.com/martin-1992/Java-Lock-Notes/tree/master/Java%20CAS%20%E5%8E%9F%E7%90%86)
　　CAS（compare and swap），从英文全称可知有两步，比较和交换，在不使用锁的情况下，在多线程进行同步。如下有两步，比较和交换，这两步在 CPU 中是原子操作，即通过一个 CPU 指令（cmpxchg）完成，这样才能保证线程安全。

- 比较，CAS 将内存位置处的数值与预期数值相比较，相等，表示没有被其他线程修改；
- 交换，相等则将内存位置处的值替换为新值。

　　CAS 用在数据库中，则是乐观锁，即在数据库中多加一个字段版本号。

### [Java 上下文切换](https://github.com/martin-1992/Java-Lock-Notes/tree/master/Java%20%E4%B8%8A%E4%B8%8B%E6%96%87%E5%88%87%E6%8D%A2)
　　CPU 允许每个线程执行一段时间，然后重新选择一个线程来执行，这个时间通常是几十毫秒。比如线程 A 执行了 30 毫秒，然后切换到线程 B 执行 30 毫秒，再切换到线程 C 等，通过不停切换线程来达到多线程的效果。<br />
　　程序计数器，保证能切换回上一个线程中的操作。这种上下文最适合 IO 密集型的，比如 A 线程执行时需要等待 IO 返回，这段时间可切换到其他线程执行。

### [Java 线程池源码分析](https://github.com/martin-1992/Java-Lock-Notes/tree/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90)

- execute，线程池的核心方法。主要是添加任务到阻塞队列 workQueue.offer(command) 和创建线程执行任务 addWorker；
    1. 任务执行有两种，一是直接创建线程执行，只有在线程数小于核心线程或是线程数小于最大线程且阻塞队列已满的情况下；
    2. 二是线程从阻塞队列中获取任务执行，第二种是最普遍的执行任务情况。
- addWorker，线程数 ctl 加一，将要执行的任务包装成 Worker 对象，启动线程；
    1. 使用自旋锁 + CAS 来保证线程数加一成功。这里不用锁，是因为太笨重了，只是个加一操作，使用 CAS 即可；
    2. 使用 ReentrantLock 来上锁，上锁之后，会重新检查线程池状态，类似双重检查锁，在进行操作；
    2. 使用局部变量，为虚拟机栈，属于线程私有，可保证线程安全。
- runWorker，执行任务，任务来自 Worker 中封装的 task，如果为空，则从阻塞队列中获取任务。该线程会不断循环，从阻塞队列中获取任务，直到阻塞队列为空，才会判断是否要关闭线程。

### [ThreadLocal](https://github.com/martin-1992/Java-Lock-Notes/tree/master/ThreadLocal)
　　每个线程都有各自的 Map，变量的存放和获取是对 Map 进行操作，本质是用空间副本来保证线程安全。

- 每个线程都有各自的变量 threadLocals，类型为 ThreadLoacalMap，包含线程各自的本地变量；
- ThreadLoacalMap 是由 Entry 对象的数组组成的，通过哈希值（步长为 0x61c88647，取下一个哈希值），使用位运算来获取索引下标，所以数组长度必须为 2 的次方；
- Entry 对象，为弱引用（WeakReferences）。是由 Key 和 value 组成，key 为 ThreadLocal，value 为当前线程要存的本地变量值。因为是 Entry 是弱引用，存在 key 为 null（被 GC 回收了），但 value 不为 null，所以需要手动回收，防止内存泄漏。

### [happens-before 规则](https://github.com/martin-1992/Java-Lock-Notes/tree/master/happens-before%20%E8%A7%84%E5%88%99)
　　表示前面一个操作的结果对后续操作是可见的。编译器优化会对 CPU 指令进行重排序，**而 happens-before 约束了编译器的优化行为，即要求编译器的优化要遵守 happens-before 规则。**

- **程序顺序规则。** 一个线程中的每个操作，happens-before 于该线程中的任意后续操作，即按照程序顺序执行，先执行第一行代码，在执行第二行代码，以此类推；
- **监视器锁规则。** 对一个锁的解锁，happens-before 于随后对这个锁的加锁。以 synchronized 关键字为例，它提供自动加锁和解锁，线程 A 在 synchronized 修饰的代码块中，将 x 更新为 12。执行完该代码块后，会自动解锁，然后线程 B 进入该代码块，能看到 x=12；
- **volatile 变量规则。** 对一个 volatile 域的写，happens-before  于任意（线程）后续对这个 volatile 的读。假设有两个线程，线程 A 读一个 volatile 变量，线程 B 写同一个 volatile 变量，则线程 B 的写操作是在线程 A 之前的；
- **传递性规则。** 如果 A happens-before B，且 B happens-before C，那么 A happens-before C；
- **start() 规则。** 如果线程 A 执行操作 ThreadB.start() （启动线程 B），那么 A 线程的 ThreadB.start() 操作 happens-before 于线程 B 中的任意操作；
- **join() 规则。** 如果线程 A 执行操作 ThreadB.join() 并成功返回，那么线程 B 中的任意操作 happens-before 于线程 A 从 ThreadB.join() 操作成功返回。

### [volatile 原理分析](https://github.com/martin-1992/Java-Lock-Notes/tree/master/volatile%20%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90)
　　volatile 修饰的共享变量在进行写操作时会使用 lock 命令（隐式的，在汇编代码中），其作用是：

- 将当前 CPU 缓存的数据写回到电脑内存，而不是先写到写缓冲区中；
- 使用缓存一致性协议，其它 CPU 检查该内存地址是否被修改，修改则重新到内存中获取最新的数据。

### [死锁问题及解决方法](https://github.com/martin-1992/Java-Lock-Notes/tree/master/%E6%AD%BB%E9%94%81%E9%97%AE%E9%A2%98%E5%8F%8A%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95)
　　破坏以下任意一个条件，即可解决死锁问题。

- 互斥，共享资源 X 和 Y 只能被一个线程占用；
- 占有且等待，比如线程 T1 获得共享资源 X，会不释放该共享资源，继续等待共享资源 Y；
- 不可抢占，其它线程不能强行抢占线程 T1 占有的资源；
- 循环等待，线程 T1 等待线程 T2 释放资源，线程 T2 等待线程 T1 释放资源。