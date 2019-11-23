### remove
　　从对象 Entry 数组中移除某个 key（ThreadLoacal）和对应的变量值。

- 获取该 key（ThreadLocal）的哈希值，使用位运算获取索引下标；
- 从当前索引 i 开始遍历，因为可能遇到哈希冲突，使用线性探测法；
- 找到 key，则将 key 的软引用设为 null；
- 调用 [expungeStaleEntry]()，清除所有 key 为 null 的 Entry。

```java
    private void remove(ThreadLocal<?> key) {
        Entry[] tab = table;
        int len = tab.length;
        // 获取该 key（ThreadLocal）的哈希值，使用位运算获取索引下标
        int i = key.threadLocalHashCode & (len-1);
        // 从当前索引 i 开始遍历，因为可能遇到哈希冲突，使用线性探测法
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            // 找到 key，则清除
            if (e.get() == key) {
                // 将 key 的软引用设为 null
                e.clear();
                // 清理所有 key 为 null 的 Entry
                expungeStaleEntry(i);
                return;
            }
        }
    }
```