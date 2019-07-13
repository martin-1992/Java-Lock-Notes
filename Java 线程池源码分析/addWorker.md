
## addWorker
　　为要执行的任务创建 Worker 对象，并调用 Worker 对象的 thread 属性，启动线程来处理阻塞队列里的任务。
  
- 获取 ctl 和线程池的当前状态，判断线程池是否还在工作、符合创建线程条件；
- 如线程池满足条件，还在工作，则使用自旋，判断线程池中的有效线程数是否小于最大容量，小于则使用 CAS 创建线程（自旋保证创建成功）；
- 将任务包装成 Worker 对象，添加到 hashset 中，启动线程，即调用 Worker 对象的 thread 属性，处理阻塞队列里的任务。

```java
    private final HashSet<Worker> workers = new HashSet<Worker>();
    
    private boolean addWorker(Runnable firstTask, boolean core) {
    
        // retry 是一个无限循环，当线程池处于 RUNNING 状态时，只有在线程池中
        // 的有效线程数被成功加一以后，才会退出该循环而去执行下面的代码。即当
        // 线程池在 RUNNING 状态下退出该 retry 循环时，线程池中的有效线程数一
        // 定少于此次设定的最大线程数（可能是 corePoolSize 或 maximumPoolSize）
        retry:
        for (;;) {
            // 获取 ctl 和线程池的当前状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 将该判断转换成 rs >= SHUTDOWN && (rs != SHUTDOWN ||
            // firstTask != null || workQueue.isEmpty)，线程池满足以下一种
            // 条件，则结束该循环，返回 false，即判断线程池是否还在工作：
            // 线程池处于 STOP，TYDING 或 TERMINATD 状态；
            // 不在 RUNNING 状态，firstTask 不为空，即线程池接受了新的任务；
            // 不在 RUNNING 状态，阻塞队列 workQueue 为空。
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
            
            // 使用自旋，保证创建线程成功
            for (;;) {
                // 线程池中的有效线程数
                int wc = workerCountOf(c);
                // 有效线程数大于最大容量 CAPACITY，或大于 maximumPoolSize / corePoolSize，则返回 false。
                // 当 core 为 true 时，则使用 corePoolSize 来判断，false 使用 maximumPoolSize
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 使用 CAS 操作线程池，线程数量加一，CAS 成功的话则跳出循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 如果线程池的状态改变了，则重新循环操作
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        
        // 任务是否成功启动标识
        boolean workerStarted = false;
        // 任务是否成功添加标识
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 根据参数 firstTask 来创建 Worker 对象
            w = new Worker(firstTask);
            // 创建线程对象
            final Thread t = w.thread;
            // ThreadFactory 构造出的 Thread 有可能是 null，做个判断
            if (t != null) {
                // 获取锁，并上锁
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // 在锁住之后再重新检测一下检查线程池的状态
                    int rs = runStateOf(ctl.get());
                    // 线程池的状态为运行状态，或为关闭状态且 firstTask 为空时
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 如线程已启动，则抛出异常
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 将该 Worker 对象添加到 hashset 中
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        // 标识一下任务已经添加成功
                        workerAdded = true;
                    }
                } finally {
                    // 解锁
                    mainLock.unlock();
                }
                if (workerAdded) {
                    // 启动线程，这里的 t 是 Worker 中的 thread 属性，所以相当于就是调用了 Worker 的 run 方法    
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            // worker 启动失败，则回收该 worker
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

### addWorkerFailed
　　当添加 worker 失败时，则会上锁回收 worker：
    
- 从 workers 中删除掉对应的 worker；
- workCount 减一。

```java
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }
    
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }
```
