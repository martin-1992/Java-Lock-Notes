
## tryTerminate
　　检查线程池是否满足终止运行条件，是则终止线程池的运行，不是则线程池继续运行。

- 判断线程池是否满足终止条件，不终止的情况：1、线程池还在运行。2、线程池已经在终止运行中。3、线程池已经关闭，但阻塞队列还有任务要处理；
- 当线程池不在运行状态，阻塞队列也没有任务，则判断是否还有 Worker 处于运行状态，如还有运行，则调用 [interruptIdleWorkers](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/interruptIdleWorkers.md) 方法中断没有在运行任务的 Worker 并回收；
- 回收完闲置 Worker 后，上锁使用 CAS，设置线程池的状态为清理状态 TIDYING；
- 调用 terminalted 方法，将状态变为 TERMINATED，解锁。

```java
    private static final boolean ONLY_ONE = true;

    protected void terminated() { }

    private final Condition termination = mainLock.newCondition();

    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 满足以下条件，则不终止线程池的运行：1、线程池还在运行；2、线程池处于 TIDYING
            // 或 TERMINATED 状态，说明已经在关闭了，不允许继续处理；3、线程池处于
            // SHUTDOWN 状态并且阻塞队列不为空，这时候还需要处理阻塞队列的任务，不能终止
            // 线程池
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 线程池不在运行状态，阻塞队列也没有任务，但还有 Worker 处于运行状态
            if (workerCountOf(c) != 0) { // Eligible to terminate
                // 中断一个闲置 Worker 线程，Worker 会回收，然后还是会调用 tryTerminate 
                // 方法，如果还有闲置线程，那么继续中断
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 使用 CAS，将线程池状态转为清理状态 TIDYING
                try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        // 调用 terminalted 方法
                        terminated();
                    } finally {
                        // terminated 方法调用完毕之后，状态变为 TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                // 解锁
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```
