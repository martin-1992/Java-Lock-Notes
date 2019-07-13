
### rs 和 wc 组成 ctl

　　ctl 可理解为单词 control 的简写，即控制，在下面代码中会简写为 c，是对线程池的运行状态和池子中的有效线程的数量进行控制的一个字段。ctl 是一个 AtomicInteger 对象，它的操作都是原子化，保证线程安全。<br />
　　一个 ctl 包含两部分信息：

- rs（runState），线程池的运行状态，用 int 变量的高 3 位表示（int 为 32 位的二进制）；
- wc（workerCount），线程池内有效线程的数量，用 int 变量的 低 29 位表示。

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    // 32 位的高 3 位表示线程池的运行状态，32 - 3 = 29
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 32 位的低 29 位表示线程池内有效线程的数量，即 2^29 - 1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
```

　　由于 ctl 变量是由 rs（线程池的运行状态）和 wc（线程池内有效线程的数量）这两个组合而成，所以，知道这两个值，就可以使用 ctlOf() 方法 计算出 ctl 的值。同理，这三个值，只要知道其中两个值，即可求出另外一个：
  
```java
    // 计算 rs，线程池的运行状态，高 3 位是 1，低 29 位是 0 的一个 int 型的数
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 计算 wc，线程池内有效线程的数量，CAPACITY 为 2^29 - 1，高 3 位是 0，低 29 位是 1 的一个 int 型的数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    // 计算 ctl
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

### 线程池的运行状态

- RUNNING（运行状态），能接受新提交的任务，并且也能处理阻塞队列中的任务；
- SHUTDOWN（关闭状态），不接受新提交的任务，但可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态时，调用 shutdown() 方法会使线程池进入到该状态。当然，finalize() 方法在执行过程中或许也会隐式地进入该状态；
- STOP，不能接受新提交的任务，也不能处理阻塞队列中已保存的任务，并且会中断正在处理中的任务。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
- TIDYING（清理状态），所有任务都执行完，且当前线程池已没有有效的线程，这个时候线程池的状态将会 TIDYING，将调用 terminated() 方法。当线程池处于 SHUTDOWN 状态时，如果此后线程池内没有线程了并且阻塞队列内也没有待执行的任务了，线程池就会进入到该状态。当线程池处于 STOP 状态时，如果此后线程池内没有线程了，线程池就会进入到该状态；
- TERMINATED（终止状态），terminated() 方法执行完后就进入该状态。


```java
    // runState is stored in the high-order bits
    // -1 << 29 = -536870912
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 0 << 29 = 0
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 1 << 29 = 536870912，即 2^29
    private static final int STOP       =  1 << COUNT_BITS;
    // 2 << 29 = 1073741824，即 2^30
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 3 << 29 = 1610612736
    private static final int TERMINATED =  3 << COUNT_BITS;
```


　　上面五个常量是按照从小到大的属性排列的，可通过比较大小判断出属于哪个状态。
  

```java
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
    // 
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
```

### 构造函数 ThreadPoolExecutor
　　ThreadPoolExecutor 有四个构造函数，但其实都是调用同一个构造函数。
  
```java
    // 核心线程数
    private volatile int corePoolSize;
    // 最大线程数
    private volatile int maximumPoolSize;
    // 阻塞队列
    private final BlockingQueue<Runnable> workQueue;
    // 线程的空闲等待时间
    private volatile long keepAliveTime;
    // 线程工厂
    private volatile ThreadFactory threadFactory;
    // 拒绝策略
    private volatile RejectedExecutionHandler handler;

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        // 参数检验，不符则抛出异常
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        // 赋值语句，将各参数分别保存到该类内部的 6 个成员字段
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```  

#### corePoolSize
　　核心线程，表示线程池中一直存活的最小线程数量。默认情况下，核心线程是按需创建启动的，只有当线程池接收到任务请求后，才会去创建并启动一定数量的核心线程来执行任务。而如果没有接收到相关任务，则不会主动创建核心线程，有助于降低系统资源的消耗。也可调用 prestartCoreThread() 或 prestartAllCoreThreads() 方法，在初始时线程池就创建启动一个或所有核心线程。
  
#### maximumPoolSize
　　线程池内能够容纳线程数量的最大值，最大不能超过 CAPACITY 值，即 1 << 29 - 1。如果将线程池的核心线程数 corePoolSize 和最大线程数 maximumPoolSize 设置为相同的数值，即线程池中的所有线程都是核心线程，那么该线程池就是一个容量固定的线程池。当通过方法 execute(Runnable) 提交一个任务到线程池时，会判断运行状态（RUNNING）的线程数量与核心线程数（corePoolSize）的大小：
  
- 运行状态的线程数少于核心线程数，那么即使有一些非核心线程处于空闲等待状态，系统也会倾向于创建一个新的线程来处理这个任务；
- 运行状态的线程数大于核心线程数，但小于最大线程数，系统会先判断线程池内部的阻塞队列 workQueue 中是否还有空位，如果发现有空位，系统会将该任务先存入阻塞队列。没空位，即队列已满，则会创建一个线程来执行该任务。  

#### keepAliveTime
　　表示空闲线程处于等待状态的超时时间，超过该时间，该线程就会停止工作。

- 当 allowCoreThreadTimeOut 设为 false，总线程数大于核心线程数时，多出来的非核心线程一旦进入到空闲等待状态，会开始计算各自的等待时间，超过 keepAliveTime 时，该线程就会停止工作（terminated）。核心线程不受此限制，即使超时，也不会停止工作；
- 当 allowCoreThreadTimeOut 设为 true 时，无论是非核心线程还是核心线程，一旦超时，就会停止工作。

#### workQueue
　　是一个内部元素为 Runnable（各种任务，通常是异步的任务） 的阻塞队列 BlockingQueue。阻塞队列是一种类似于“生产者-消费者”模型的队列，当队列已满时，如果继续向队列中插入元素，该插入操作将会被阻塞一直处于等待状态，直到队列中有元素被移除产生空位后，才能执行插入操作。当队列为空时，如果继续执行元素的删除或获取操作，该操作同样会被被阻塞而进入等待状态，直到队列中又有了该元素后，才有可能执行该操作。
  
#### threadFactory
　　线程工厂，用于创建线程。创建线程池的时候未指定 threadFactory，则默认使用 Executors.defaultThreadFactory() 方法来创建线程工厂。
  
#### handler  
　　拒绝策略。以下两个条件满足其中任意一个的时候，如果继续向该线程池中提交新的任务，线程池将会调用 handler 的 rejectedExecution 方法，表示拒绝执行这些新提交的任务：

- 当线程池处于 SHUTDOWN（关闭）状态时，无论线程池和阻塞队列是否都已满；
- 当线程池中的所有线程都处于运行状态并且线程池中的阻塞队列已满时。



### execute
　　创建 Worker 对象，为执行任务的线程。
  
- 如果当前正在执行的 Worker 数量比 corePoolSize（核心线程）要小，调用 [addWorker](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/addWorker.md) 方法，直接创建一个新的 Worker 处理执行阻塞队列中的任务；
- 如果当前正在执行的 Worker 数量大于等于 corePoolSize，将任务放到阻塞队列里，如果阻塞队列没满并且状态是 RUNNING 的话，直接丢到阻塞队列，否则执行第3步。
    1. 丢到阻塞队列之后，还需要再做一次验证（丢到阻塞队列之后可能另外一个线程关闭了线程池或者刚刚加入到队列的线程死了），如果这个时候线程池不在 RUNNING 状态，把刚刚丢入队列的任务 remove 掉，调用 reject 方法；
    2. 否则查看 Worker 数量，如果 Worker 数量为0，起一个新的 Worker 去阻塞队列里拿任务执行；
- 丢到阻塞失败的话，会调用 addWorker 方法尝试起一个新的 Worker 去阻塞队列拿任务并执行任务，如果这个新的 Worker 创建失败，调用 reject 方法。


```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        // 获取 ctl 的值
        int c = ctl.get();
        // 如果线程池中的有效线程数小于核心线程数，调用 addWorker 方法创建一个新的 Worker 来执行任务，
        // 这里 true 为使用核心线程的数量，false 使用最大线程数量
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //线程池的有效线程数大于核心线程数，线程池在 Running 状态，阻塞队列也没满（使用 offer 方法来判断，
        // true 表示队列没满成功添加，false 表示队列已满添加失败）
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 再对线程池做一次判断，防止出现线程突然关闭的情况，如果线程不在 Running 状态，
            // 使用 remove 方法移除掉刚刚入队列的任务，调用 reject 方法对新任务执行该拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 线程池的有效数量为 0，创建 Worker 去阻塞队列里拿任务执行
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // addWorker 创建 Worker 失败，则调用 reject 方法，false 为最大线程池的大小
        else if (!addWorker(command, false))
            reject(command);
    }
```

#### remove

- 从队列移除该任务；
- [tryTerminate()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/tryTerminate.md)，检查线程池是否满足终止运行条件，是则终止线程池的运行，不是则线程池继续运行。

```java
    public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }
```
