---
title: java高并发详解-11-线程上下文类加载器
copyright: true
comments: true
related_posts: true
date: 2021-03-08 15:16:35
tags: 线程上下文类加载器
categories: 高并发详解
---

前面讲java类加载器的知识为了解释线程的上下文类加载器原理和使用场景

# 线程上下文类加载器



## 为什么需要线程上下文类加载器

根据Thread 类的文档 你会发现线程上下文方法是  从JDK1.2 开始引入的，getContextClassLoader() 和 setContextClassLoader（ClassLoader cl）分别用于获取和设置当前线程线程的上下文类加载器，如果当前线程没有设置上下文类加载器，那么它将和父线程保持同样的类加载器。

- 站在开发者的角度，其他线程都是由Main线程，也就是main函数所在的线程派生的，它是其他线程的父线程或者祖先线程。

```java
public class MainThreadClassLoader {
    public static void main(String[] args) throws ClassNotFoundException {
        System.out.println(currentThread().getContextClassLoader());
        Class.forName("com.mysql.jdbc.Driver");
    }
}
运行结果:
sun.misc.Launcher$AppClassLoader@14dad5dc
    
    Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.
```



为什么要有线程上下文类加载器呢，这就与**JVM类加载器双亲委托机制自身的缺陷有关。**

- jdk的核心库中提供了很多SPI （Service Provider Interface），常见的SPI 包括JDBC、JCE、JNDI、JAXP 和JBI 等，JDK只规定了这些接口之间的逻辑关系，但不提供具体的实现，具体的实现需要由 第三方厂商来提供，
- 作为Java程序员 都写过JDBC的程序，在编写JDBC程序时几乎百分之百的都在与 java.sql 包下的类打交道



如下图所示，Java使用JDBC这个SPI 完全透明了 应用程序和第三方厂商数据库驱动的具体实现， 不管数据库类型如何切换，应用程序只需要替换JDBC 的驱动jar包以及数据库的驱动名称即可，而不用进行任何更新 。

![](/uploads/java-concurrency-master/JDBC_SPI.png)

这样做的好处是：

- JDBC 提供了高度抽象，应用程序只需要面向接口编程即可，不用关心各大数据厂商的具体实现 。
- 但是问题在于 java.lang.sql 中的所有接口都是JDK 提供，**加载这些接口的类加载器是 根加载器**,  但是 **第三方厂商提供的类库驱动 是由系统类加载器加载的，**
- 由于JVM 类加载器的双亲委托机制，比如 Connections、 Statement、 RowSet 等 都是由 **根加载器加载**，第三方的JDBC 驱动包中的实现不会被加载 。

通过分析 Mysql数据库的源码，来看看是如果解决这个 接口与实现类的 加载器不一致的问题

## 数据库驱动的初始化源码分析



在编写所有的 JDBC程序时，首先都需要 调用Class.forName("xxxx.xxxx.xxxx.Driver")对数据库驱动进行加载，打开 Mysql驱动 Driver源码，代码如清单11-2 所示

### Driver源码

- 这个时老版本的 （com.mysql.jdbc.Driver）

```java
/**
 * Backwards compatibility to support apps that call <code>Class.forName("com.mysql.jdbc.Driver");</code>.
 */
public class Driver extends com.mysql.cj.jdbc.Driver {
    public Driver() throws SQLException {
        super();
    }

    static {
        System.err.println("Loading class `com.mysql.jdbc.Driver'. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver'. "
                + "The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary.");
    }
}
```

- 这个是新版本的，推荐使用的（com.mysql.cj.jdbc.Driver）

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    //
    // Register ourselves with the DriverManager
    //
    static {
        try {
            // 在Mysql 的静态方法中 将Driver 实例注册到DriverManager中 
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }

    /**
     * Construct a new driver and register it with DriverManager
     * 
     * @throws SQLException
     *             if a database error occurs.
     */
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```

- Driver类的静态代码块主要是 将Mysql 的Driver实例注册给 DriverManager，因此直接使用 DriverManager.registerDriver（new com.mysql.jdbc.Driver（））其作用与 Class.forName ("xxx.xxx.xxx.Driver")是完全等价的 



### DriverManager源码

```java
  //  Worker method called by the public getConnection() methods.
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
        /*
         * When callerCl is null, we should check the application's
         * (which is invoking this class indirectly)
         * classloader, so that the JDBC driver class outside rt.jar
         * can be loaded from here.
         */
        // 注释 1
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }
//..................
// 注释 2
        for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info);
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }
        }
.。。。。。。。
    }
     
        
 // 注释 3 
private static boolean isDriverAllowed(Driver driver, ClassLoader classLoader) {
        boolean result = false;
        if(driver != null) {
            Class<?> aClass = null;
            try {
                aClass =  Class.forName(driver.getClass().getName(), true, classLoader);
            } catch (Exception ex) {
                result = false;
            }

             result = ( aClass == driver.getClass() ) ? true : false;
        }

        return result;
    }
```

1. 在注释1处 获取当前线程的上下文类加载器 ，该类就是调用Class.forName("X") 所在线程的线程上下文类加载器，通常是系统类加载器
2. 注释2 中通过递归DriverManager 中已经注册的驱动类，然后验证 该数据库驱动 是否可以被指定的类加载器加载（线程上下文类加载器），如果验证通过，则返回Connection，此刻返回的 Connection 则是数据库厂商提供的实例
3. 注释3 关键地方在于Class.forName(driver.getClass().getName(), true, classLoader);  其使用线程上下文类加载器及逆行数据库驱动的加载以及初始化  



总结一下数据库驱动加载的整个过程，

- 由于JDK 定义了SPI的标准接口，加之这些接口被作为 JDK 核心标准类库的一部分，既想要完全透明标准接口的实现，又想与JDK 核心库进行捆绑， 由于JVM 类加载器双亲委托机制的限制，启动类加载器不可能 加载得到第三方厂商提供的具体实现 。
-  为了解决这一问题，JDK 只好提供一种不太优雅的设计-线程上下文类加载器 
- 有了线程上下文类加载器，启动类加载器（根加载器）反倒需要委托子类加载器去加载厂商提供的SPI 具体实现。父委托变成了子委托的方式



# 本章总结

- 分析Mysql驱动加载过程的源码，清晰地理解线程上下文加载器所发挥地作用了 

- 在Thread 类中增加 getContextClassLoader() 和 setContextClassLoader（ClassLoader cl）方法实属无奈之举，它不仅破坏了类加载器地父委托机制，而且反其道行之，允许“子委托机制”，