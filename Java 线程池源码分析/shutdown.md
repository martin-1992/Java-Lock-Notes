
## shutdown
　　终止线程池的运行。
　　
- checkShutdownAccess()，首先检查关闭线程池的权限；
- advanceRunState(SHUTDOWN)，把线程池状态更新为 SHUTDOWN；
- interruptIdleWorkers，遍历 Worker 集合，中断空闲的 Worker；
- onShutdown()，钩子方法，实现为空，默认不处理；
- tryTerminate，检查线程池是否满足终止运行的条件，是则终止线程池的运行。

```java
    void onShutdown() {
    }

    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        // 上锁
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            // 解锁
            mainLock.unlock();
        }
        // 检查线程池是否满足终止运行的条件，是则终止线程池的运行
        tryTerminate();
    }
```

### checkShutdownAccess
　　检查是否有关闭线程池的权限。

```java
    private static final RuntimePermission shutdownPerm = new RuntimePermission("modifyThread");

    private void checkShutdownAccess() {
        // 获取权限管理
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            // 检查权限
            security.checkPermission(shutdownPerm);
            final ReentrantLock mainLock = this.mainLock;
            // 上锁
            mainLock.lock();
            try {
                // 检查 Worker 集合的 worker
                for (Worker w : workers)
                    security.checkAccess(w.thread);
            } finally {
                mainLock.unlock();
            }
        }
    }
```

### advanceRunState
　　把线程池状态更新为 SHUTDOWN。

```java
    private void advanceRunState(int targetState) {
        for (;;) {
            // 获取线程池的状态
            int c = ctl.get();
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                break;
        }
    }
```
