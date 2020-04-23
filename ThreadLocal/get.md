### get

- 获取当前线程绑定的 ThreadLocalMap，key 为当前访问的 ThreadLocal 对象，从该 key 获取对应的 Entry；
- 如果 ThreadLocalMap 为空，则调用初始化，该方法会调用用户自定义方法 initialValue，即添加用户自定义的初始值。

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

#### getMap
　　线程对应的 ThreadLocalMap，注意，ThreadLocalMap 是线程绑定的。即每个线程都有一个 ThreadLocalMap，保证线程安全。

```java
    ThreadLocal.ThreadLocalMap threadLocals = null;

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
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

#### createMap
　　创建线程对应的 [ThreadLocalMap](https://github.com/martin-1992/Java-Lock-Notes/tree/master/ThreadLocal/ThreadLocalMap)。

```java 
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
