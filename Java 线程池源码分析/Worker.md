### 构造函数
　　Worker 为线程池 ThreadPoolExecutor 内的工作线程，继承了 AQS 的类，使用 AQS 的独占锁方法。这里不用重入锁 ReentrantLock，**是为了实现不可重入的特性的去反应线程现在的执行状态。** <br />
　　因为不可重入，所以 state 只有两种状态，0 为没获取到锁，1 为获取到锁。如果是可重入锁，state 则表示锁的计数。构造函数中传入 Runnalbe 参数为任务，使用线程工厂创建线程来执行该任务。

```java
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable{

        private static final long serialVersionUID = 6138294804551838833L;

        /** 该 Worker 正在运行的线程 thread，如果为空，表示线程工厂创建 thread 失败 */
        final Thread thread;
        /** 初始化要运行的任务，可为空 */
        Runnable firstTask;
        /** 计数器，表示这个 Worker 完成的任务数 */
        volatile long completedTasks;

        /**
         * 使用 threadFactory 创建 Thread，Thread 内部的 Runnable 就是 Worker，所以得到 Worker 的 thread 并
         * start 的时候，会执行 Worker 的 run 方法，也就是执行 ThreadPoolExecutor 的 runWorker 方法
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            // 把状态位设置成 -1，这样任何线程都不能得到 Worker 的锁，除非调用了 unlock 方法
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            // 使用 ThreadFactory 创建新线程 
            this.thread = getThreadFactory().newThread(this);
        }
}
```

### run
　　Worker 类重写了 run 方法，使用 ThreadPoolExecutor#runWorker 方法（在 addWorker 方法里调用），直接启动 Worker 的话，会调用 ThreadPoolExecutor 的 runWork 方法。<br />
　　需要特别注意的是这个 Worker 是实现了 Runnable 接口的，thread 线程属性使用 ThreadFactory 构造 Thread 的时候，构造的 Thread 中使用的 Runnable 其实就是 Worker。

```java
    public void run() {
        runWorker(this);
    }
```        

### isHeldExclusively
　　获取锁的状态，0 为没有获取锁，1 为获取锁。

```java
    protected boolean isHeldExclusively() {
        return getState() != 0;
    }
```

### tryAcquire
　　使用 CAS 来获取锁，获取到锁，则锁的状态为 1。

```java
    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }
```

### tryRelease
　　释放锁，设置锁状态为 0。

```java
    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }
```

### interruptIfStarted
　　中断线程运行，线程池在执行 shutdownNow() 时会调用该方法。

```java
    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
```
