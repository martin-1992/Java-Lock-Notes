## ThreadPoolExecutor 分析

### [ThreadPoolExecutor](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/ThreadPoolExecutor.md)
　　ThreadPoolExecutor 有四个构造函数，但其实都是调用同一个构造函数。各参数讲解：

- **corePoolSize，核心线程。** 表示线程池中一直存活的最小线程数量；
- **maximumPoolSize，最大线程数。** 最大不能超过 CAPACITY 值，即 1 << 29 - 1；
- **keepAliveTime，空闲线程的等待时间。** 超过该时间，该线程就会停止工作；
- **workQueue，阻塞队列。** 当线程池中的核心线程都在处理任务，新任务（Runnable）会先添加到阻塞队列中；
- **threadFactory，线程工厂。** 用于创建线程，默认使用 Executors.defaultThreadFactory() 来创建线程工厂；
- **handler，拒绝策略。** 调用 handler 的 rejectedExecution 方法，用于拒绝新任务执行。

### [execute](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/execute.md)
　　线程池的核心方法，主要是**添加任务到阻塞队列 workQueue.offer(command) 和创建线程执行任务 addWorker。** <br />
　　任务执行有两种，一是直接创建线程执行，只有在线程数小于核心线程或是线程数小于最大线程且阻塞队列已满的情况下。二是线程从阻塞队列中获取任务执行，第二种是最普遍的执行任务情况。

### [addWorker](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/addWorker.md)
　　使用 CAS 让线程数加一，包装成 Worker 对象添加到线程池 workers，**启动 worker 对象上的线程 thread。**

### [runWorker](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/runWorker.md)
　　执行任务，任务来自 Worker 中封装的 task，如果为空，则从阻塞队列中获取任务。**该线程会不断循环，从阻塞队列中获取任务，** 直到阻塞队列为空，才会判断是否要关闭线程。

### reference

- [Java线程池实现原理及其在美团业务中的实践](https://mp.weixin.qq.com/s/baYuX8aCwQ9PP6k7TDl2Ww)
- [Java 线程池 ThreadPoolExecutor 源码分析](https://blog.csdn.net/cleverGump/article/details/50688008)
- [深度解读 java 线程池设计思想及源码实现](https://juejin.im/entry/59b232ee6fb9a0248d25139a)
