## thread_pool_executor_analysis


### [ThreadPoolExecutor](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/ThreadPoolExecutor.md)
　　ThreadPoolExecutor 有四个构造函数，但其实都是调用同一个构造函数。各参数讲解：

- corePoolSize，核心线程，表示线程池中一直存活的最小线程数量；
- maximumPoolSize，线程池内能够容纳的最大线程数量，最大不能超过 CAPACITY 值，即 1 << 29 - 1；
- keepAliveTime，表示空闲线程处于等待状态的超时时间，超过该时间，该线程就会停止工作；
- workQueue，是一个内部元素为 Runnable（各种任务，通常是异步的任务） 的阻塞队列 BlockingQueue，创建一个线程后从阻塞队列获取任务并执行；
- threadFactory，线程工厂，用于创建线程；
- handler，拒绝新任务执行的策略。

### [Worker](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/Worker.md)
　　Worker 是一个继承 AQS 的类，可调用 AQS 的使用独占锁方法（同一时刻仅有一个线程去获取同步状态（锁））。构造函数中传入 Runnalbe 参数，为任务，使用 线程工厂创建线程来执行该任务。<br />
　　Worker 类重写了 run 方法，使用 ThreadPoolExecutor 的 runWorker 方法（在 addWorker 方法里调用），直接启动 Worker 的话，会调用 ThreadPoolExecutor 的 runWork 方法。需要特别注意的是这个 Worker 是实现了 Runnable 接口的，thread 线程属性使用 ThreadFactory 构造 Thread 的时候，构造的 Thread 中使用的 Runnable 其实就是 Worker。

### [getTask](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/getTask.md)
　　先检查是否需要回收多余或空闲的 Worker 线程，再从阻塞队列 workQueue 中获取任务：

- Worker 个数大于线程池的最大线程数，线程数减一；
- 线程池处于关闭状态 SHUTDOWN，且阻塞队列为空，表示没有任务在进行，线程数减一；
- 线程池处于 STOP 状态，线程数减一；
- Worker 数量大于最大线程数 maximumPoolSize；
- 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)。

### [addWorker](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/addWorker.md)
　　为要执行的任务创建 Worker 对象，并调用 Worker 对象的 thread 属性，启动线程来处理阻塞队列里的任务。

- 获取 ctl 和线程池的当前状态，判断线程池是否还在工作、符合创建线程条件；
- 如线程池满足条件，还在工作，则使用自旋，判断线程池中的有效线程数是否小于最大容量，小于则使用 CAS 创建线程（自旋保证创建成功）；
- 将任务包装成 Worker 对象，添加到 hashset 中，启动线程，即调用 Worker 对象的 thread 属性，处理阻塞队列里的任务。

### [runWorker](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/runWorker.md)
　　Worker.run() 方法调用的是 ThreadPoolExecutor 的 runWorker 方法，执行任务，任务来自 Worker 中封装的 task，如为空，则从阻塞队列中获取任务。

- 获取当前线程，获取 Worker 中封装的任务 task；
- 检查线程池是否在运行，不在运行则中断当前线程，取消任务的执行；
- 线程池在运行，判断 Worker 中的任务是否为空，为空则调用 getTask 从阻塞队列中获取任务，然后对 Worker 上锁，调用 task.run() 使用当前线程执行任务；
- 任务执行完后，记录该 Worker 执行任务的个数，解锁，将 Worker 变为闲置 Worker；
- [processWorkerExit()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/processWorkerExit.md)，调用该方法回收 Worker，通过 completedAbruptly 变量来判断是否正常结束。

### [processWorkerExit()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/processWorkerExit.md)
　　判断 Worker 是否正常结束，非正常结束则进行处理。

- 判断 Worker 是否正常结束，非正常结束调用 decrementWorkerCount；
- 从线程池的 Worker 集合移除掉该 Worker，进行回收；
- [tryTerminate()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/tryTerminate.md)，检查线程池是否满足条件，是则终止线程池的运行；
- 如 Worker 处于正常运行，且 Worker 是正常结束。判断 Worker 数量与核心线程数量的大小关系，大于则不需要创建 Worker。当 Worker 执行任务出现异常，Worker 数量小于核心线程的数量，新创建一个 Worker 替代原先的 Worker。

### [tryTerminate()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/tryTerminate.md)
　　检查线程池是否满足终止运行条件，是则终止线程池的运行，不是则线程池继续运行。

- 判断线程池是否满足终止条件，不终止的情况：1、线程池还在运行。2、线程池已经在终止运行中。3、线程池已经关闭，但阻塞队列还有任务要处理；
- 当线程池不在运行状态，阻塞队列也没有任务，则判断是否还有 Worker 处于运行状态，如还有运行，则调用 [interruptIdleWorkers](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/interruptIdleWorkers.md) 方法中断没有在运行任务的 Worker 并回收；
- 回收完闲置 Worker 后，上锁使用 CAS，设置线程池的状态为清理状态 TIDYING；
- 调用 terminalted 方法，将状态变为 TERMINATED，解锁。

### [interruptIdleWorkers](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/interruptIdleWorkers.md)
　　遍历 Worker 集合，中断闲置的 Worker，根据是否能获取 Worker 的锁和是否可以中断来判断是否为闲置 Worker。
  
### [shutdown](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/shutdown.md)
　　终止线程池的运行。
　　
- checkShutdownAccess()，首先检查关闭线程池的权限；
- advanceRunState(SHUTDOWN)，把线程池状态更新为 SHUTDOWN；
- [interruptIdleWorkers()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/interruptIdleWorkers.md)，遍历 Worker 集合，中断空闲的 Worker；
- onShutdown()，钩子方法，实现为空，默认不处理；
- [tryTerminate()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/tryTerminate.md)，检查线程池是否满足终止运行的条件，是则终止线程池的运行。

### reference
- https://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/
- https://blog.csdn.net/cleverGump/article/details/50688008
- https://blog.csdn.net/programmer_at/article/details/79799267
- https://juejin.im/entry/59b232ee6fb9a0248d25139a
- https://www.imooc.com/article/42990
- https://www.jianshu.com/p/65da61177eed
