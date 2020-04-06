### runWorker
　　Worker.run() 方法调用的是 ThreadPoolExecutor#runWorker 方法，执行任务，任务来自 Worker 中封装的 task，如为空，则从阻塞队列中获取任务。
  
- 获取当前线程，获取 Worker 中封装的任务 task；
- 如果线程 worker 中没有任务，则使用 getTask() 从阻塞队列中获取任务；
- 检查线程池是否在运行，不在运行则中断当前线程，取消任务的执行；
- 线程池在运行，调用 task.run() 使用当前线程执行任务；
- 任务执行完后，记录该 Worker 执行任务的个数，解锁，将 Worker 变为闲置 Worker； 
- 当 getTask() 为空时，即阻塞队列中没任务执行，则调用 [processWorkerExit()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/processWorkerExit.md) 回收线程 Worker，通过 completedAbruptly 变量来判断是否正常结束。

```java
    final void runWorker(Worker w) {
        // 获取当前线程
        Thread wt = Thread.currentThread();
        // 获取 Worker 中封装的任务 task
        Runnable task = w.firstTask;
        // 将 Worker 中的任务置空
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 如果 Worker 中任务 task 不为空，则上锁。如果为空，则使用 getTask 从阻塞队列中获得任务，
            // 这里使用循环，直到获得任务为止，或者是任务为空才退出
            while (task != null || (task = getTask()) != null) {
                // 获取到任务后，则对 Worker 上锁，表示当前 Worker 开始执行任务
                w.lock();
                // 执行任务前先检查线程池的状态：
                // 1. 线程池处于 STOP 状态，并且当前线程 wt 没有被中断，则中断线程 wt，不执行 Worker 任务；
                // 2. 线程池处于 RUNNING 或 SHUTDOWN 状态，并且当前线程已被中断，使用 runStateAtLeast
                // 重新检查线程池状态，如果是处于 STOP 状态且没有被中断，则中断线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    // 抽象方法，子类继承实现，任务执行前做什么
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        // 真正开始执行任务
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        // 抽象方法，子类继承实现，任务结束后做什么
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 执行完，设置 task 为 null，用于 GC 回收
                    task = null;
                    // 记录该线程 Worker 执行任务的个数
                    w.completedTasks++;
                    // 任务执行完后，解锁，Worker 变成闲置 Worker
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 回收线程 Worker 方法
            processWorkerExit(w, completedAbruptly);
        }
    }
```

### getTask
　　从阻塞队列 workQueue 中获取任务。
  
- timed 为 true，使用 pool 方法获取任务，若等待一定的时间取不到任务，则返回 null；
- timed 为 false，使用 take 方法一直阻塞等待阻塞队列新进数据。

```java
    private Runnable getTask() {
        // 如果使用超时时间并且也没有拿到任务的标识
        boolean timedOut = false;

        for (;;) {
            // 获取 ctl 和线程池的状态
            int c = ctl.get();
            int rs = runStateOf(c);
            
            // 线程池是关闭状态 SHUTDOWN，且阻塞队列为空，表示没有任务在进行，则 worker
            // 数量减一，返回 null。SHUTDOWN 状态还会处理阻塞队列的任务，但阻塞队列为空就
            // 结束了。如果线程池是 STOP 状态，则 worker 数量减一，返回 null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            
            // 获取线程池内的有效线程数量，即 worker 个数
            int wc = workerCountOf(c);

            // allowCoreThreadTimeOut 默认为 false，即线程池中的核心线程在闲置状态也不会被回收。
            // 如果 worker 数大于核心线程数，即有部分为非核心线程，则需要回收 worker。如果 
            // allowCoreThreadTimeOut 为 true，表示核心线程超时 keepAliveTime 也会被回收   
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 满足以下条件，则 worker 数减一，返回 null，之后会进行 Worker 回收工作：
            // worker 数大于最大线程数，并且 worker 数大于 1或阻塞队列为空
            // 超时，且 worker 数大于 1 或阻塞队列为空
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // timed 为 true，使用 pool 方法获取任务，若等待一定的时间取不到任务，则返回 null
                // timed 为 false，使用 take 方法一直阻塞等待阻塞队列新进数据
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

