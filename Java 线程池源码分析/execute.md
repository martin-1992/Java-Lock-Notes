### execute
　　线程池的核心方法，主要是**添加任务到阻塞队列 workQueue.offer(command) 和创建线程执行任务 [addWorker](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/addWorker.md)。**

- workerCount < corePoolSize，则创建一个核心线程来执行新任务 addWorker(command, true)；
- workerCount > corePoolSize，且阻塞队列 workQueue 没有满，则将任务添加到阻塞队列中，workQueue.offer(command) 添加成功；
- workerCount >= corePoolSize && workerCount < maximumPoolSize，且阻塞队列已满，则创建一个线程来执行新任务，addWorker(command, false)；
- workerCount >= maximumPoolSize，且阻塞队列已满，则执行 handler 的拒绝策略来处理该任务, 默认的处理方式是直接抛异常，reject(command)。

```java
    private final BlockingQueue<Runnable> workQueue;

    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        // 获取 ctl 的值
        int c = ctl.get();
        // 如果线程池中的有效线程数小于 corePoolSize，调用 addWorker 方法创建线程来执行任务，直接返回
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            // 创建线程失败，检查线程池的状态，看是否还在运行
            c = ctl.get();
        }
        // 线程池的有效线程数大于核心线程数，线程池在 Running 状态，阻塞队列也没满（使用 offer 方法来判断，
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
        // 到这里时，阻塞队列已满，尝试创建线程 addWorker 执行任务，如果创建失败，则执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```
