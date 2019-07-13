
## Worker
　　Worker 是一个继承 AQS 的类，可调用 AQS 的使用独占锁方法（同一时刻仅有一个线程去获取同步状态（锁））。构造函数中传入 Runnalbe 参数，为任务，使用 线程工厂创建线程来执行该任务。<br />
　　Worker 类重写了 run 方法，使用 ThreadPoolExecutor 的 runWorker 方法（在 addWorker 方法里调用），直接启动 Worker 的话，会调用 ThreadPoolExecutor 的 runWork 方法。需要特别注意的是这个 Worker 是实现了 Runnable 接口的，thread 线程属性使用 ThreadFactory 构造 Thread 的时候，构造的 Thread 中使用的 Runnable 其实就是 Worker。

```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** 该 Worker 正在运行的线程 thread，如果为空，表示线程工厂创建 thread 失败 */
        final Thread thread;
        /** 初始化要运行的任务，可为空 */
        Runnable firstTask;
        /** 计数器，表示这个 Worker 完成的任务数 */
        volatile long completedTasks;

        /**
         * 使用 hreadFactory 创建 Thread，Thread 内部的 Runnable 就是 Worker，所以得到 Worker 的 thread 并
         * start 的时候，会执行 Worker 的 run 方法，也就是执行 ThreadPoolExecutor 的 runWorker 方法
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            // 把状态位设置成 -1，这样任何线程都不能得到Worker的锁，除非调用了unlock方法
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 使用 ThreadFactory 创建新线程 
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.
        // 获取锁的状态，0 为没有获取锁，1 为获取锁
        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```
