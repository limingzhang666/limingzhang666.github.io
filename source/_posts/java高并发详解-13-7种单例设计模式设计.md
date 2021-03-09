---
title: java高并发详解-13-7种单例设计模式设计
copyright: true
comments: true
related_posts: true
date: 2021-03-09 14:00:40
tags: 单例设计模式设计
categories: 高并发详解

---

## 饿汉式

```java
public final class HungerSingleton {

    private HungerSingleton() {
    }

    private static HungerSingleton instance = new HungerSingleton();

    public static HungerSingleton getInstance() {
        return instance;
    }
}
```

- instance 作为类变量在 类初始化的过程中会被收集进 <clinit>()方法中，该方法能够百分之百地保证同步，也就是说 instance 在多线程地情况下不可能被实例化两次，但是instance 被ClassLoader 加载后可能很长一段时间才被使用，那就意味着 instance 实例所开辟地堆内存会驻留更久地时间
- 如果一个类中地成员属性比较少，且占用的内存资源不多，饿汉的方式也未尝不可， 
- 相反，如果一个类中的成员都是比较重的资源，那么这种方式就会有些不妥
- 总结起来，饿汉式的单例设计模式可以保证在多线程的情况下的唯一实例，getInstance的性能也比较高，
- 但是无法进行懒加载



## 懒汉式+同步方法

所谓懒汉式就是在使用类实例的时候再去创建，这样就可以避免在类初始化时，提前创建，

```java

public final class LazySingleton {
    
    private LazySingleton() {
    }

    private static LazySingleton instance = null;

    public static synchronized LazySingleton getInstance() {
        //如果不加synchronized， 会出现多线程并发问题,
        //加上 synchronized后，单线程，效率问题
        if (instance == null) {
            instance = new LazySingleton();
        }
        return instance;
    }
}
```

- 注意看 getInstance ()方法，如果此处不加 synchronized 同步，则可能存在多线程并发的问题，
- 但是加上 synchronized后，单线程，有可能存在效率问题



## Volatile+Double-Check

```java
public final class VolatileDoubleheckSingleton {
    
    private volatile static VolatileDoubleheckSingleton instance = null;
    Connection conn;
    Socket socket;

    private VolatileDoubleheckSingleton() {
        // this.conn;
        //this.socket;
    }

    public static VolatileDoubleheckSingleton getInstance() {
        if (instance == null) {
            synchronized (VolatileDoubleheckSingleton.class) {
                if (instance == null) {
                    instance = new VolatileDoubleheckSingleton();
                }
            }
        }
        return instance;
    }
}
```

- Double-Check 虽然是一种巧妙地程序设计，但是有可能引起类成员变量地实例化 conn 和socket 发生在instance 实例化之后，（因为JVM在运行时指令重排序导致的）
- 而 volatile关键字则可以防止这种重排序的发生 所以 private volatile static VolatileDoubleheckSingleton instance = null;



## Holder方式

Holder 的方式完全是 借助了类加载的特点，

```java
public final class HolderSingleton {
    private HolderSingleton() {
    }

    // 在静态内部类中持有 HolderSingleton 的实例，并且可以直接初始化
    private static class Holder {
        private static HolderSingleton instance = new HolderSingleton();
    }

    // 调用 getInstance 方法，事实上是获得 Holder的instance 静态属性
    public static HolderSingleton getInstance() {
        return Holder.instance;
    }
}

```

好处：

- 在HolderSingleton类中 没有instance的静态成员，而是将其放到了静态内部类Holder之中，因此在Singleton类的 初始化过程中 并不会创建 Singleton 的实例，
- Holder类中定义了Singleton的静态变量，并且直接进行了实例化。当Holder被主动引用的时候  则会创建Singleton的实例，Singleton实例的创建过程 在Java程序编译时期 收集至 <clinit>（） 方法中，该方法 又是同步方法， 同步方法可以保证内存的可见性，JVM指令的顺序性 和原子性、
- Holder方式的单例设计是最好的设计之一



## 枚举方式

- 使用枚举的方式实现单例模式 是 《effective java》作者力推的方式 
- 枚举类型不允许被继承，同样的是线程安全的且 只能被实例化一次，但是 枚举类型不能够懒加载，

```java


public enum EnumSingleton {
    INSTANCE;

    EnumSingleton() {
        System.out.println("INIT");
    }

    public static void method() {
        // 调用该方法则会主动使用Singleton INSTANCE 将会被实例化
    }

    public static EnumSingleton getInstance() {
        return INSTANCE;
    }
}



public class Singleton {
    //实例变量
    private byte[] data = new byte[1024];

    private Singleton() {
    }

    private enum EnumHolder {
        INSTANCE;
        private Singleton instance;

        EnumHolder() {
            this.instance = new Singleton();
        }

        private Singleton getSingleton() {
            return instance;
        }
    }

    public static Singleton getInstance() {
        return EnumHolder.INSTANCE.getSingleton();
    }
}
```

