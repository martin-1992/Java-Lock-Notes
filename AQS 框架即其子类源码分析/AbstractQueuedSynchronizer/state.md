### 同步状态 state
　　AQS 使用一个 int 类型的成员变量 state 来表示同步状态，即用于同步线程之间的共享状态，通过 CAS 和 volatile 保证其原子性和可见性。**使用 state 可实现重入锁，同一个锁对象可多次获取锁，一定程度上避免死锁。**
  
- state > 0 时，已经获取了锁；
    1. 可重入锁。判断 state != 0，即有线程获取锁，然后判断是否是当前线程获取锁，是则执行 state + 1；
    2. 非可重入锁。判断 state != 0，则有线程获取锁，进入阻塞状态；
- state = 0 时，释放了锁。
    1. 可重入锁，释放时执行 state - 1，当 state = 0，表示该线程真正释放了锁；
    2. 非可重入锁，直接将 state = 0，释放锁。

　　对同步状态 state 进行操作的方法有 getState、setState、compareAndSetState。

```java
/**
 * 同步状态
 */
private volatile int state;

/**
 * 返回当前的同步状态值
 */
protected final int getState() {
    return state;
}

/**
 * 设置新的同步状态值
 */
protected final void setState(int newState) {
    state = newState;
}

/**
 * 使用 CAS 来尝试获取锁，比如，compareAndSetState(0, 1) 为尝试获取锁，并将同步
 * 状态从 0 更新为 1，前面提到同步状态为 0 时，表示该锁已释放。
 */
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```