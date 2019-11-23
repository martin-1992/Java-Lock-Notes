### rehash

- 调用 [expungeStaleEntry]()，清除所有 key 为 null 的 Entry；
- 如果 size + threshold / 4 >= threshold，则扩容。

```java
    private void rehash() {
        // 清除所有 key 为 null 的 Entry
        expungeStaleEntries();

        // 等价于 size + threshold / 4 >= threshold，进行扩容
        if (size >= threshold - threshold / 4)
            resize();
    }
```

#### resize
　　两倍扩容，将旧数组的值重新哈希，赋值到新数组中。

```java
    private void resize() {
        // 使用变量 oldTab 保存当前 Entry 数组
        Entry[] oldTab = table;
        int oldLen = oldTab.length;
        // 两倍扩容
        int newLen = oldLen * 2;
        Entry[] newTab = new Entry[newLen];
        int count = 0;
        
        for (int j = 0; j < oldLen; ++j) {
            Entry e = oldTab[j];
            if (e != null) {
                ThreadLocal<?> k = e.get();
                // 旧 Entry 数组的 key（ThreadLocal）已经被 GC 回收了（弱引用），则将值置为空（
                // 因为不是弱引用，所以需手动置为空），用于 GC 回收
                if (k == null) {
                    e.value = null; 
                } else {
                    // 对 key 重新哈希，获取新的索引下标，赋值到新的数组中
                    int h = k.threadLocalHashCode & (newLen - 1);
                    while (newTab[h] != null)
                        // 哈希冲突，找到下个索引进行赋值
                        h = nextIndex(h, newLen);
                    newTab[h] = e;
                    count++;
                }
            }
        }
        // 设置新的 Entry 数组的负载值
        setThreshold(newLen);
        size = count;
        table = newTab;
    }
```