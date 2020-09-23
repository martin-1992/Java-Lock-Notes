### 双重检查锁
　　使用 volatile 禁止指令重排序。

```java
public class Singleton {

    private volatile static Singleton singleton;

    private Singleton() {

    }

    public static Singleton getSingleton() {
        if (singleton == null) {
            synchronized (Singleton.class) {
                if (singleton == null) {
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }

}
```

### 懒汉式
　　第一次调用才初始化，避免内存浪费。

```java
public class Singleton {

    private static Singleton instance;

    private Singleton() {

    }

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

### 饿汉式
　　类加锁，浪费内存。

```java
public class Singleton {

    private static Singleton instance = new Singleton();

    private Singleton() {

    }

    private staitc Singleton getInstance() {
        return instance;
    }

}
```

### 登记式 / 静态内部类
　　适用于静态域的情况，只有显式调用 getInstance 方法时，才会加载 SingletonHolder 类，从而实例化。

```java
public class Singleton {

    private static class SingletonHolder {
        private static final Singleton INSTANCE = new Singleton();
    }

    private Singleton() {

    }

    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

### 枚举
　　自动支持序列化机制，防止反序列化重新创建新的对象和多次实例化。

```java
public class Singleton {

    public staitc void main(String[] args) {
        Singleton singleton = Singleton.INSTANCE;
    }

    enum Singleton {
        INSTANCE;

        private Singleton() {

        }
    }
}
```

### reference

- [单例模式](https://www.runoob.com/design-pattern/singleton-pattern.html)




