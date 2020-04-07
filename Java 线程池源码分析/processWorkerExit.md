### processWorkerExit
　　该方法是在 [runWorker()](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/runWorker.md) 中调用的，将线程 worker 从线程池 workers 中移除，让 JVM 去回收。

- 判断 Worker 是否正常结束，非正常结束 completedAbruptly=true 则调用 decrementWorkerCount，线程数 ctl 减一；
- 上锁，从线程池 workers 移除掉该线程 Worker，进行回收；
- [tryTerminate()](https://github.com/martin-1992/Java-Lock-Notes/blob/master/Java%20%E7%BA%BF%E7%A8%8B%E6%B1%A0%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/tryTerminate.md)，检查线程池是否满足条件，是则终止线程池的运行；
- 如 Worker 处于正常运行，且 Worker 是正常结束。判断线程 Worker 数量与核心线程数量的大小关系，大于则不需要创建 Worker。当线程数小于核心线程的数量，新创建一个线程 worker。

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 在 runWorker 中，如果没有执行任务，则 completedAbruptly 为 true，线程数 ctl 减一，
        // 只有当执行了任务，completedAbruptly 为 false，线程数不减
        if (completedAbruptly)
            decrementWorkerCount();
        
        // 上锁，从线程池 workers 中移除掉指定线程 worker
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 加上 Worker 完成的任务数，计算总的完成任务数
            completedTaskCount += w.completedTasks;
            // 从线程池的 Worker 集合移除掉需要回收的 Worker
            workers.remove(w);
        } finally {
            // 解锁
            mainLock.unlock();
        }
        
        // 检查线程池是否满足条件，是则终止线程池的运行
        tryTerminate();
        // 获取线程池的状态
        int c = ctl.get();
        // 如果线程池处于 RUNNING 或者 SHUTDOWN 状态
        if (runStateLessThan(c, STOP)) {
            // Worker 是正常流程结束的
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                // 核心线程为 0，阻塞队列不为空，设置线程最小为 1
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果 Worker 数量大于核心线程的数量，则不需要创建 Worker
                if (workerCountOf(c) >= min)
                    return;
            }
            // 当 Worker 执行任务出现异常，Worker 数量小于核心线程的数量，
            // 新创建一个 Worker 替代原先的 Worker
            addWorker(null, false);
        }
    }
```

### decrementWorkerCount
　　使用 CAS 使得线程数 ctl 减一。

```java
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }
    
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }
```
