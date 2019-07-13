
## runWorker
　　Worker.run() 方法调用的是 ThreadPoolExecutor 的 runWorker 方法，执行任务，任务来自 Worker 中封装的 task，如为空，则从阻塞队列中获取任务。
  
- 获取当前线程，获取 Worker 中封装的任务 task；
- 检查线程池是否在运行，不在运行则中断当前线程，取消任务的执行；
- 线程池在运行，判断 Worker 中的任务是否为空，为空则调用 getTask 从阻塞队列中获取任务，然后对 Worker 上锁，调用 task.run() 使用当前线程执行任务；
- 任务执行完后，记录该 Worker 执行任务的个数，解锁，将 Worker 变为闲置 Worker； 
- [processWorkerExit()](https://github.com/martin-1992/thread_pool_executor_analysis/blob/master/processWorkerExit.md)，调用该方法回收 Worker，通过 completedAbruptly 变量来判断是否正常结束。

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
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
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
                    // 任务执行前做什么，子类继承实现
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
                        // // 任务结束后做什么，子类继承实现
                        afterExecute(task, thrown);
                    }
                } finally {
                    // 用于 GC 回收
                    task = null;
                    // 记录该 Worker 执行任务的个数
                    w.completedTasks++;
                    // 任务执行完后，解锁，Worker 变成闲置 Worker
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // 回收 Worker 方法
            processWorkerExit(w, completedAbruptly);
        }
    }
```
