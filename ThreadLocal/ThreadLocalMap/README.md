### ThreadLocalMap
　　由多个 Entry 对象组成的数组，即 Entry[]，一个 Entry 对应一个 ThreadLocal。

- 初始大小为 2 的幂，默认为 16，可使用位运算获取下标；
- 哈希冲突时，使用线性探测法，找下个索引值下标；
- 负载因子为 2 / 3，达到 2 / 3的容量进行扩容。

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    /**
     * 初始容量，需为 2 的次方，这样能使用位运算获取索引下标
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * 底层为 Entry 的数组
     */
    private Entry[] table;

    /**
     * 数组中 Entry 的数量
     */
    private int size = 0;

    /**
     * 负载值，当达到该值，则进行扩容
     */
    private int threshold;

    /**
     * 调整负载值，负载因子为 2 / 3
     */
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }

    /**
     * 哈希冲突时，使用线性探测法，找下个索引值下标
     */
    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }

    /**
     * 前面一个索引值下标
     */
    private static int prevIndex(int i, int len) {
        return ((i - 1 >= 0) ? i - 1 : len - 1);
    }

    /**
     * 构造函数
     */
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        // 创建一个容量为 16 的 Entry 数组
        table = new Entry[INITIAL_CAPACITY];
        // 获取当前线程 ThreadLocal 的哈希值，使用位运算获取索引下标 
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        // 赋值，将 ThreadLocal 和变量值包装成 Entry
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        // 设置要扩容的阈值
        setThreshold(INITIAL_CAPACITY);
    }

    /**
     * 构造函数，将旧的 ThreadLocalMap 的值重新哈希，复制到新的 table 中
     */
    private ThreadLocalMap(ThreadLocalMap parentMap) {
        // 获取该 ThreadLocalMap 的 Entry 数组
        Entry[] parentTable = parentMap.table;
        int len = parentTable.length;
        // 根据旧 Entry 数组的大小，设置新的负载值
        setThreshold(len);
        // 创建新的 Entry 数组
        table = new Entry[len];
        
        // 遍历，将旧 Entry 数组的值重新哈希，赋值到新 Entry 数组中
        for (int j = 0; j < len; j++) {
            // 旧 Entry 数组的值
            Entry e = parentTable[j];
            if (e != null) {
                @SuppressWarnings("unchecked")
                ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                if (key != null) {
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    // 旧 Entry 数组的值重新哈希
                    int h = key.threadLocalHashCode & (len - 1);
                    // 哈希冲突，使用线性探测法，找到下个索引下标
                    while (table[h] != null)
                        h = nextIndex(h, len);
                    table[h] = c;
                    size++;
                }
            }
        }
    }
}
 ```
