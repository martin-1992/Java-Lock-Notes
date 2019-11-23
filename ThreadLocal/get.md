### get

- 获取当前线程副本 ThreadLocalMap 中当前线程对应的变量值，保证线程安全；
- 如果线程副本为空，则调用初始化，该方法会调用用户自定义方法 initialValue，即添加用户自定义的初始值。

```java
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 获取该线程对应的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 根据当前线程的 threadLocal（key）获取 Entry，保存了变量值
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

#### setInitialValue
　　获取当前线程的 ThreadLocalMap，调用 [ThreadLoacalMap#set](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/set.md)，根据当前线程 key（ThreadLoacal） 从 ThreadLoacalMap 设置该 key 对应的变量值，变量值为用户自定义的初始值。

```java
    private T setInitialValue() {
        // 用户重写该方法，获取自定义的初始值
        T value = initialValue();
        Thread t = Thread.currentThread();
        // 获取当前线程对应的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    
    protected T initialValue() {
        return null;
    }
```

#### getMap
　　线程对应的 ThreadLocalMap。

```java
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

#### createMap
　　创建线程对应的 [ThreadLocalMap](https://github.com/martin-1992/Java-Lock-Notes/tree/master/ThreadLocal/ThreadLocalMap)。

```java 
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```