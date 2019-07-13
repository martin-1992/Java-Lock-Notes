
## getTask
　　先检查是否需要回收多余或空闲的 Worker 线程，从阻塞队列 workQueue 中获取任务：
  
- Worker 个数大于线程池的最大线程数，线程数减一；
- 线程池处于关闭状态 SHUTDOWN，且阻塞队列为空，表示没有任务在进行，线程数减一；
- 线程池处于 STOP 状态，线程数减一；
- worker 数量大于最大线程数 maximumPoolSize；
- 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)。

```java
    private Runnable getTask() {
        // 如果使用超时时间并且也没有拿到任务的标识
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            // 获取 ctl 和线程池的状态
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 线程池是关闭状态 SHUTDOWN，且阻塞队列为空，表示没有任务在进行，则 worker
            // 数量减一，返回 null。SHUTDOWN 状态还会处理阻塞队列的任务，但阻塞队列为空就
            // 结束了。如果线程池是 STOP 状态，则 worker 数量减一，返回 null
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }
            
            // 获取线程池内的有效线程数量，即 worker 个数
            int wc = workerCountOf(c);

            // Are workers subject to culling?
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
