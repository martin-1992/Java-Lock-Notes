### set
　　获取当前线程的 ThreadLocalMap，调用 [ThreadLoacalMap#set](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/set.md)，根据当前线程 key（ThreadLoacal） 从 ThreadLoacalMap 设置该 key 对应的变量值。

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
　　创建线程对应的 [ThreadLocalMap](https://github.com/martin-1992/Java-Lock-Notes/tree/master/ThreadLocal/ThreadLocalMap)。

```java 
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```