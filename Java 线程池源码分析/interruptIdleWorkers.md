
## interruptIdleWorkers
　　遍历 Worker 集合，中断闲置的 Worker，根据是否能获取 Worker 的锁和是否可以中断来判断是否为闲置 Worker。

```java
    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private void interruptIdleWorkers(boolean onlyOne) {
        
        final ReentrantLock mainLock = this.mainLock;
        // 中断闲置 Worker 需要加锁，防止并发
        mainLock.lock();
        try {
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
                // 为 true，则中断一个 Worker 后就退出，否则遍历所有 Worker 对象，尝试中断每个 Worker 对象
                if (onlyOne)
                    break;
            }
        } finally {
            // 解锁
            mainLock.unlock();
        }
    }
```
