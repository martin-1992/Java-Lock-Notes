### CAS 概念
　　CAS（compare and swap），从英文全称可知有两步，比较和交换，在不使用锁的情况下，在多线程进行同步。如下有两步，**比较和交换，这两步在 CPU 中是原子操作，** 即通过一个 CPU 指令（cmpxchg）完成，这样才能保证线程安全。

- 比较，CAS 将内存位置处的数值与预期数值相比较，相等，表示没有被其他线程修改；
- 交换，相等则将内存位置处的值替换为新值。

### 背景介绍
　　CPU 是通过总线和内存进行数据传输的。多个核心通过同一条总线和内存以及其他硬件进行通信，如下图（图片来自深入理解计算机系统）：

![avatar](photo_1.jpg)

　　上图中，CPU 通过两个蓝色箭头标注的总线与内存进行通信。如果不能保证同步机制，则在多线程下会出现并发问题（类似出现数据库的脏读、幻读问题），本质是多个线程对同一个内存进行操作导致的。<br />
　　CPU 中提供了 lock，让处理器可以独占使用某些共享内存。比如将 lock 添加在 ADD 指令前，可让该指令具备原子性。即多个核心执行该指令，会以串行方式进行。在 Intel 处理器，有两种方法保证处理器的某个核心（线程）独占某片内存区域。

- **锁定总线。** 让某个核心（线程）独占使用总线，这会导致其它核心无法访问，类似 synchronize 这样的重量级锁；
- **锁定缓存。** 如果某处内存数据被缓存在处理器缓存中，则锁定缓存行对应的内存区域。同样其他核心无法访问，但相比锁总线，代价小很多。

### AtomicInteger
　　AtomicInteger 为原子类，即它的方法是原子性的，通过封装 unsafe.compareAndSwapInt 来调用。其它原子类也是同理，这里选取一个来分析。unsafe 是底层，其核心代码是是 CPU 中一条带 lock 前缀的 cmpxchg 指令，即“比较并交换”指令，不相等子则自旋（while 循环）进行重试，直到成功为止。

- unsafe，获取并操作内存的数据；
- valueOffset，存储 value 在 AtomicInteger 中的偏移量；
- value，存储 AtomicInteger 的 int 值，该属性需要借助 volatile 关键字保证其在线程间是可见的。

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
    
    private volatile int value;
    
    // 预期值和要更新的值
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
    
    // ......
}

public final class Unsafe {
    // compareAndSwapInt 是 native 类型的方法
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
    // ......
}
```

### CAS 的缺点

- **ABA 问题。** 假设当前值从 A -> B -> A，而 CAS 根据预期值为 A，会判定没有改变，进行更新。ABA 问题的解决方案，是在变量前加上版本号，即 1A -> 2B -> 3A，这样 CAS 就能判断出 1A != 3A；
- **自旋（循环）时间长，开销大。** CAS 更新是使用 while (true) 循环，直到更新成功打破循环。如果一直重试失败，会不断自旋，给 CPU 带来开销；
- **只能保证一个共享变量的原子操作。** CAS 无法对多个变量使用。但是 Java 从 1.5 开始 JDK 提供了 AtomicReference 类来保证引用对象之间的原子性，可以把多个变量放在一个对象里来进行CAS操作。

### reference

- [Java CAS 原理分析](https://www.tianxiaobo.com/2018/05/15/Java-%E4%B8%AD%E7%9A%84-CAS-%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)
- [【基本功】不可不说的Java“锁”事](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651749434&idx=3&sn=5ffa63ad47fe166f2f1a9f604ed10091&chksm=bd12a5778a652c61509d9e718ab086ff27ad8768586ea9b38c3dcf9e017a8e49bcae3df9bcc8&scene=21#wechat_redirect)
