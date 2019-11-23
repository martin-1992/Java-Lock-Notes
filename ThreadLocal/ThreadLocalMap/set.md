### set
　　从对象 Entry 数组中设置某个 key（ThreadLoacal）及其对应的变量值。

- 根据要查询 key 的哈希值，使用位运算获取索引下标；
- 哈希没冲突，key 相同，则新值覆盖旧值，返回；
- 哈希冲突，key 不相同，则寻找下个索引下标值，重复步骤二的流程；
- 如果遇到 key 为 null，则清除；
- 如果没有可清除的 Entry，且数组长度达到负载值，则调用 [rehash](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/rehash.md) 扩容。

```java
    private void set(ThreadLocal<?> key, Object value) {
        // 获取 Entry 数组
        Entry[] tab = table;
        int len = tab.length;
        // 计算要查询 key（ThreadLocal） 的哈希值，使用位运算获取索引下标 
        int i = key.threadLocalHashCode & (len-1);
        // 从获取的索引下标开始遍历，因为可能遇到哈希冲突，使用了线性探测法
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();
            // key 相同，新值覆盖旧值
            if (k == key) {
                e.value = value;
                return;
            }
            // key 为空，表示被 GC 回收了，因为 key 是软引用 WeakReferences（软引用下次 GC 会回收），
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        // 该 key 没有对应的 Entry，则创建一个新的 Entry
        tab[i] = new Entry(key, value);
        int sz = ++size;
        // 没有可清除的 Entry，且数组长度达到负载值，则扩容
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
```

#### replaceStaleEntry
　　找到 key，则新值替换旧值。没找到，则重新生成一个 Entry 对象，添加到数组中。遇到 key 为 null，调用 [expungeStaleEntry](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/expungeStaleEntry.md) 进行清除。

```java
    private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                   int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;

        // 在 set 方法中调用，staleSlot 为索引下标
        int slotToExpunge = staleSlot;
        for (int i = prevIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = prevIndex(i, len))
            if (e.get() == null)
                slotToExpunge = i;

        // Find either the key or trailing null slot of run, whichever
        // occurs first
        for (int i = nextIndex(staleSlot, len);
             (e = tab[i]) != null;
             i = nextIndex(i, len)) {
            ThreadLocal<?> k = e.get();

            // 找到 key，则替换
            if (k == key) {
                e.value = value;

                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;

                // Start expunge at preceding stale entry if it exists
                if (slotToExpunge == staleSlot)
                    slotToExpunge = i;
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }

            // If we didn't find stale entry on backward scan, the
            // first stale entry seen while scanning for key is the
            // first still present in the run.
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }

        // 没找到 key，则新建一个 Entry 对象，添加
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);

        // If there are any other stale entries in run, expunge them
        if (slotToExpunge != staleSlot)
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    }
```

#### cleanSomeSlots
　　遍历，调用 [expungeStaleEntries](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/expungeStaleEntry.md) 清除那些 key 为 null，但 Entry 不为 null。

```java
    private boolean cleanSomeSlots(int i, int n) {
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        do {
            i = nextIndex(i, len);
            Entry e = tab[i];
            if (e != null && e.get() == null) {
                n = len;
                removed = true;
                i = expungeStaleEntry(i);
            }
        } while ( (n >>>= 1) != 0);
        return removed;
    }
```