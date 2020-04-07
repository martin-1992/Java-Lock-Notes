### tryTerminate
　　检查线程池是否满足终止运行条件，是则终止线程池的运行，对线程池中的每个线程 worker 尝试终止。

- 判断线程池是否满足终止条件，以下条件则不终止，直接返回；
    1. 线程池还在运行；
    2. 线程池已经在终止运行中；
    3. 线程池已经关闭，但阻塞队列还有任务要处理。
- 当线程池不在运行状态，阻塞队列也没有任务，则判断是否还有线程 Worker 处于运行状态，如有则调用 interruptIdleWorkers 方法中断没有在运行任务的 Worker 并回收；
- 回收完闲置 Worker 后，上锁使用 CAS，设置线程池的状态为清理状态 TIDYING；
- 调用 terminalted 方法，将状态变为 TERMINATED，解锁。

```java
    private static final boolean ONLY_ONE = true;

    protected void terminated() { }
    
    private final Condition termination = mainLock.newCondition();

    final void tryTerminate() {
        for (;;) {
            // 获取线程池状态
            int c = ctl.get();
            // 以下条件，不终止线程池的运行：
            // 1、线程池还在运行；
            // 2、线程池处于 TIDYING 或 TERMINATED 状态，标志已经在关闭了；
            // 3、线程池处于 SHUTDOWN 状态并且阻塞队列不为空，这时候还需要处理阻塞队列的任务，不能终止线程池
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 线程池不在运行状态，阻塞队列也没有任务，但还有线程 worker 处于运行状态
            if (workerCountOf(c) != 0) {
                // 中断一个闲置 Worker 线程，Worker 会回收，然后还是会调用 tryTerminate 
                // 方法，如果还有闲置线程，那么继续中断
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 使用 CAS，将线程池状态设置为清理状态 TIDYING
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

### interruptIdleWorkers
　　遍历线程池中的线程，中断闲置的线程 worker，根据是否能获取线程 worker 的锁和是否可以中断来判断是否为闲置线程 Worker。

```java
    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 遍历线程池中的线程
            for (Worker w : workers) {
                // 获取 worker 中的线程
                Thread t = w.thread;
                // Worker 中的线程没有被中断，且 Worker 可以获取锁，为闲置 Worker。
                // 如果不能获取锁，说明 Worker 在执行任务，不中断
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        // 中断闲置 Worker 线程
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        // 释放 Worker 锁
                        w.unlock();
                    }
                }
                // 为 true，则中断一个 Worker 后就退出，否则遍历所有 Worker 对象，尝试中断空闲的（没有获取锁的） Worker 对象
                if (onlyOne)
                    break;
            }
        } finally {
            // 解锁
            mainLock.unlock();
        }
    }
```
