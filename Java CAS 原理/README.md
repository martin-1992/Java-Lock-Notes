### CAS 概念
　　CAS（compare and swap），从英文全称可知有两步，比较和交换。是一种用于在多线程环境下实现同步功能的机制。

- 比较，CAS 将内存位置处的数值与预期数值相比较，相等，表示没有被其他线程修改；
- 交换，相等则将内存位置处的值替换为新值。

### 背景介绍
　　CPU 是通过总线和内存进行数据传输的。多个核心通过同一条总线和内存以及其他硬件进行通信，如下图（图片来自深入理解计算机系统）：

![avatar](photo_1.jpg)

　　上图中，CPU 通过两个蓝色箭头标注的总线与内存进行通信。如果不能保证同步机制，则在多线程下会出现并发问题（类似出现数据库的脏读、幻读问题），本质是多个线程对同一个内存进行操作导致的。<br />
　　CPU 中提供了 lock，让处理器可以独占使用某些共享内存。比如将 lock 添加在 ADD 指令前，可让该指令具备原子性。即多个核心执行该指令，会以串行方式进行。在 Intel 处理器，有两种方法保证处理器的某个核心（线程）独占某片内存区域。

- 锁定总线，让某个核心（线程）独占使用总线，这会导致其它核心无法访问，类似 synchronize 这样的重量级锁；
- 锁定缓存，如果某处内存数据被缓存在处理器缓存中，则锁定缓存行对应的内存区域。同样其他核心无法访问，但相比锁总线，代价小很多。

### AtomicInteger
　　AtomicInteger 为原子类，即它的方法是原子性的，通过封装 unsafe.compareAndSwapInt 来调用。其它原子类也是同理，这里选取一个来分析。unsafe 是底层，其核心代码是是一条带 lock 前缀的 cmpxchg 指令，即“比较并交换”指令。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            // 计算变量 value 在类对象中的偏移
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
    // 预期值和要更新的值
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    // ......
}

public final class Unsafe {
    // compareAndSwapInt 是 native 类型的方法，继续往下看
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
    // ......
}
```

### reference

- [Java CAS 原理分析](https://www.tianxiaobo.com/2018/05/15/Java-%E4%B8%AD%E7%9A%84-CAS-%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)；










