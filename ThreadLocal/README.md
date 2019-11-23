### ThreadLocal
　　ThreadLocalMap 结构如下图。

- 每个线程都有各自的 ThreadLoacalMap，包含线程各自的本地变量；
- ThreadLoacalMap 是由 Entry 对象的数组组成的，通过哈希值（步长为 0x61c88647，取下一个哈希值），使用位运算来获取索引下标，所以数组长度必须为 2 的次方；
- Entry 对象，为弱引用（WeakReferences）。是由 Key 和 value 组成，key 为 ThreadLocal，value 为当前线程要存的本地变量值。因为是 Entry 是弱引用，存在 key 为 null（被 GC 回收了），但 value 不为 null，所以需要手动回收，防止内存泄漏。

![avatar](photo_1.png)

```java
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

    /**
     * 哈希值，从 0 开始
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * 步长
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * 下个哈希值，根据一定步长获取的
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    /**
     * 用户自定义的初始值，在 get 方法获取不到时，会调用该方法返回自定义的初始值
     */
    protected T initialValue() {
        return null;
    }

    public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
        return new SuppliedThreadLocal<>(supplier);
    }

    public ThreadLocal() {
    }
}
```

### [get](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/get.md)
　　获取当前线程副本 ThreadLocalMap，根据当前线程 key（ThreadLoacal）获取对应的变量值。

### [set](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/set.md)
　　获取当前线程副本 ThreadLocalMap，根据当前线程 key（ThreadLoacal）设置对应的变量值。

### [remove](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/remove.md)
　　获取当前线程副本 ThreadLocalMap，删除当前线程 key（ThreadLoacal）和其对应的变量值。
