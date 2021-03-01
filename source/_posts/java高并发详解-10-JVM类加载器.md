---
title: java高并发详解-10-JVM类加载器
copyright: true
comments: true
related_posts: true
date: 2021-03-01 22:46:10
tags: JVM类加载器
categories: 高并发详解
---

# JVM类加载器
- 类加载器就是负责类的加载的职责，对于任意一个class，都需要由加载它的类加载器和这个类本身确立其在JVM中的唯一性，这也就是 运行时包
- 任何一个对象的class 在JVM中只存在唯一的一份，比如 String.class 、Object.class 在堆内存以及方法区中肯定是唯一的 。
- 但是绝不可以理解为我们自定义的类 在JVM中同样也是这样。

## JVM内置三大类加载器

- JVM为我们提供了 三大内置的类加载器，不同的类加载器负责 将不同的类加载到JVM 内存之中，并且他们之间严格遵守着父委托的机制 

![](/uploads/java-concurrency-master/CLassLoader-father.png)



## 根类加载器介绍

根加载器又称为 Bootstrap类加载器，该类加载器是最为顶层的加载器，其没有任何父加载器，它是由 根加载器 所加载的，可以通过 -Xbootclasspath 来指定根加载器 的路径，也 可以通过系统属性来得知当前JVM 的根加载器都加载了哪些资源。

```java
public class BootStrapClassLoader
{
    public static void main(String[] args)
    {
        System.out.println("Bootstrap:" + String.class.getClassLoader());
        System.out.println(System.getProperty("sun.boot.class.path"));
    }
}
结果:
Bootstrap:null
D:\Java\Java8\jdk1.8.0\jre\lib\resources.jar;D:\Java\Java8\jdk1.8.0\jre\lib\rt.jar;D:\Java\Java8\jdk1.8.0\jre\lib\sunrsasign.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jsse.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jce.jar;D:\Java\Java8\jdk1.8.0\jre\lib\charsets.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jfr.jar;D:\Java\Java8\jdk1.8.0\jre\classes

```

