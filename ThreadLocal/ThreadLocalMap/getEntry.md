### getEntry
　　从对象 Entry 数组中获取某个 key（ThreadLoacal）中对应的变量值。

- 哈希没冲突，返回值；
- 哈希冲突，调用 getEntryAfterMiss，

```java
    private Entry getEntry(ThreadLocal<?> key) {
        // 根据该 key（ThreadLocal）的哈希值，使用位运算获取索引下标
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        // 哈希没冲突，返回值
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }
```

#### getEntryAfterMiss
　　哈希冲突时，使用线性探测法，会在当前索引，往下寻找下个值为空的索引，插入进去。<br />
　　转换为寻找该 key（ThreadLocal） 时，同样是先找到当前 key 的索引值，在往下遍历寻找符合的 key。如果遍历到 key 为 null，调用 [expungeStaleEntry]()，清除所有 key 为 null 的 Entry，防止内存泄漏。

```java
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        // 当前 Entry 数组
        Entry[] tab = table;
        int len = tab.length;
        // 从索引下标为 i 开始遍历
        while (e != null) {
            ThreadLocal<?> k = e.get();
            // 为要找的 key（ThreadLocal），直接返回
            if (k == key)
                return e;
            if (k == null)
                // key 为空，则清除该 key 对应的值，防止内存泄漏
                expungeStaleEntry(i);
            else
                // 下个索引值
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }
```

