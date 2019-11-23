### ThreadLocal

- [get](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/get.md)，获取当前线程副本 ThreadLocalMap，根据当前线程 key（ThreadLoacal）获取对应的变量值；
- [set](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/set.md)，获取当前线程副本 ThreadLocalMap，根据当前线程 key（ThreadLoacal）设置对应的变量值；
- [remove](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/remove.md)，获取当前线程副本 ThreadLocalMap，删除当前线程 key（ThreadLoacal）和其对应的变量值；

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