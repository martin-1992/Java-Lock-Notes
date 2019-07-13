
## processWorkerExit
　　判断 Worker 是否正常结束，非正常结束则进行处理。

- 判断 Worker 是否正常结束，非正常结束调用 decrementWorkerCount；
- 从线程池的 Worker 集合移除掉该 Worker，进行回收；
- [tryTerminate()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/tryTerminate.md)，检查线程池是否满足条件，是则终止线程池的运行；
- 如 Worker 处于正常运行，且 Worker 是正常结束。判断 Worker 数量与核心线程数量的大小关系，大于则不需要创建 Worker。当 Worker 执行任务出现异常，Worker 数量小于核心线程的数量，新创建一个 Worker 替代原先的 Worker。


```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果 Worker 没有正常结束流程调用 processWorkerExit 方法，Worker 数量减一。
        // 如果是正常结束的话，在 getTask 方法里 Worker 数量已减一
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();
        
        // 上锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 加上 Worker 完成的任务数，求得总的完成任务数
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
                // 核心线程为 0，Wokrer 集合不为空，设置线程最小为 1
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果 Worker 数量大于核心线程的数量，则不需要创建 Worker
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 当 Worker 执行任务出现异常，Worker 数量小于核心线程的数量，
            // 新创建一个 Worker 替代原先的 Worker
            addWorker(null, false);
        }
    }
```

### decrementWorkerCount
　　调用 CAS 使得 Worker 线程数减一。

```java
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }
```
