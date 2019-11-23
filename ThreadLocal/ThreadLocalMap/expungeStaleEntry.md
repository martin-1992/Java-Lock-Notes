### expungeStaleEntries
　　遍历 Entry 数组，如果 Entry 不为空，但 Entry 的 key（ThreadLocal）为空，则需要清除该 Entry，不清楚则 value 存在，导致内存泄漏。

```java
    private void expungeStaleEntries() {
        Entry[] tab = table;
        int len = tab.length;
        for (int j = 0; j < len; j++) {
            Entry e = tab[j];
            // Entry 不为空，但 Entry 的 key（ThreadLocal）为空，则需要清除
            // 该 Entry，不清楚则 value 存在，导致内存泄漏
            if (e != null && e.get() == null)
                expungeStaleEntry(j);
        }
    }
```

#### expungeStaleEntry
　　清除所有 key 为 null 的 Entry，即将该 key 的 value 手动赋为 null，因为 value 不为弱引用，需手动赋值，才能被 GC 回收，否则会导致内存泄漏。

- 将该 Entry 的 key 和 value 设为空，用于 GC 回收；
- 遍历下个位置的 Entry。
    1. 如果该 key 为空，则将该 Entry 的 value 设为空；
    2. 如果该 key 不为空，则重新哈希，根据新的哈希值设置新的索引下标。


```java
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;

        // 将 Entry 的 key 和 value 设为空，用于 GC 回收
        tab[staleSlot].value = null;
        tab[staleSlot] = null;
        size--;

        // Rehash until we encounter null
        Entry e;
        int i;
        for (i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();
            // key 为空，则将该 Entry 的 value 设为空
            if (k == null) {
                e.value = null;
                tab[i] = null;
                size--;
            } else {
                // key 不为空，则重新哈希
                int h = k.threadLocalHashCode & (len - 1);
                // 如果哈希的索引不等于原先的，表示是遇到哈希冲突后，使用线性探测法找到的位置 i，因为已经清除了
                // 部分 key 为 null 的 Entry，所以这里重新进行
                if (h != i) {
                    tab[i] = null;

                    while (tab[h] != null)
                        h = nextIndex(h, len);
                    tab[h] = e;
                }
            }
        }
        return i;
    }
    
    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }
```
