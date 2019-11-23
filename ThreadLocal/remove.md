### remove
　　获取当前线程的 ThreadLoacalMap，调用 [ThreadLoacalMap#remove](https://github.com/martin-1992/Java-Lock-Notes/blob/master/ThreadLocal/ThreadLocalMap/remove.md)，根据当前线程 key（ThreadLoacal） 从 ThreadLoacalMap 移除该 key 对应的变量值。

```java
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```