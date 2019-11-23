### set
　　获取当前线程的 ThreadLocalMap，调用 [ThreadLoacalMap#set]()，根据当前线程 key（ThreadLoacal） 从 ThreadLoacalMap 设置该 key 对应的变量值。

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

#### createMap
　　创建线程对应的 [ThreadLocalMap]()。

```java 
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```