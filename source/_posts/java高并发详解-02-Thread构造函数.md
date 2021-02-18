---
title: java高并发详解-02-Thread构造函数
copyright: true
related_posts: true
date: 2021-02-17 23:04:23
tags: Thread构造函数
categories: 高并发详解
---

# Thread的构造函数详解

![](/uploads/java-concurrency-master/thread-construct.png)



## 线程的命名

再构造线程的时候，推荐给线程起一个 有特殊意义的名字，这样有助于 排查问题和线程追踪。

1. ### 线程的默认命名

- Thread（）
- Thread（Runnable target）
- Thread ( ThreadGroup group , Runnable target )

源码如下：

```java
    /**
     * Allocates a new {@code Thread} object. This constructor has the same
     * effect as {@linkplain #Thread(ThreadGroup,Runnable,String) Thread}
     * {@code (null, target, gname)}, where {@code gname} is a newly generated
     * name. Automatically generated names are of the form
     * {@code "Thread-"+}<i>n</i>, where <i>n</i> is an integer.
     *
     * @param  target
     *         the object whose {@code run} method is invoked when this thread
     *         is started. If {@code null}, this classes {@code run} method does
     *         nothing.
     */
    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

    /* For autonumbering anonymous threads. */
    private static int threadInitNumber;
    private static synchronized int nextThreadNum() {
        return threadInitNumber++;
    }
```

如果没有为线程显示的指定一个名字，那么线程将会**以“Thread-” 作为前缀与 一个自增数字进行组合**，这个自增数字（threadInitNumber）在整个JVM进程中将会不断自增。

2. ###  命名线程

- Thread(Runnable target,String name)
- Thread(String name)
- Thread(ThreadGroup group,Runnable target,String name)
- Thread(ThreadGroup group,Runnable target,String name，long stackSize)
- Thread(ThreadGroup group,String name)



3. ### 修改线程的名字

不论使用的是默认的函数命名规则，还是指定了一个特殊的名字，在线程启动之前还有一个记会可以对其进行修改，一旦线程启动，名字将不再被修改。

```java
 /**
     * Changes the name of this thread to be equal to the argument
     * <code>name</code>.
     * <p>
     * First the <code>checkAccess</code> method of this thread is called
     * with no arguments. This may result in throwing a
     * <code>SecurityException</code>.
     *
     * @param      name   the new name for this thread.
     * @exception  SecurityException  if the current thread cannot modify this
     *               thread.
     * @see        #getName
     * @see        #checkAccess()
     */
    public final synchronized void setName(String name) {
        checkAccess();
        this.name = name.toCharArray();
        //线程不是NEW 状态，对其的修改将不会生效 
        if (threadStatus != 0) { 
            setNativeName(name);
        }
    }
```



## 线程的父子关系

Thread的所有构造函数，最终都会去调用一个静态方法init （）

```java

 private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name.toCharArray();
		///////////////////////////////// 获取当前线程作为父线程
        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */
            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }
            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }
        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();
        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }
        g.addUnstarted();
        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
        if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;
        /* Set thread ID */
        tid = nextThreadID();
    }
```

- Thread parent = currentThread(); 获取当前线程作为创建的父线程 。

- 线程的最初的状态是 NEW ，没有执行start方法之前，只能算一个Thread实例，并不意味着一个新的线程被创建，因此 parent 代表的将会是创建它的那个线程
- 一个线程的创建肯定是由另一个线程完成的
- 被创建线程的父线程 是创建它的线程。
- main函数 所在的线程是由 JVM创建的，也就是main线程。



## Thread 与 ThreadGroup

```java
// init 方法 片段
if (g == null) {
            /* Determine if it's an applet or not */
            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }
            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }
        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();
        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }
        g.addUnstarted();
        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
```

- 如果在构造Thread的时候没有显示指定一个ThreadGroup，那么子线程将会被加入到父线程所在的线程组。
- main线程所在的 ThreadGroup 称为 main



## Thread 与 Runnable

Thread 负责线程本身相关的职责和控制，而 Runnable 则负责逻辑执行单元的部分



## Thread与JVM虚拟机栈

 ```java
    /*
     * The requested stack size for this thread, or 0 if the creator did
     * not specify a stack size.  It is up to the VM to do whatever it
     * likes with this number; some VMs will ignore it.
     */
    private long stackSize;

  /**
     * Initializes a Thread.
     *
     * @param g the Thread group
     * @param target the object whose run() method gets called
     * @param name the name of the new Thread
     * @param stackSize the desired stack size for the new thread, or
     *        zero to indicate that this parameter is to be ignored.
     * @param acc the AccessControlContext to inherit, or
     *            AccessController.getContext() if null
     */
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
       ...
           ...
           		...
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
 ```

- stackSize=0时，代表忽略该参数 

- 一般情况下，创建线程的时候不会手动指定栈内存的地址空间字节数组，同一通过xss 参数进行设置即可
- stacksize越大，则代表着 正在线程内方法调用递归的深度越深
- stacksize越小，代表者创建的线程数量越多。
- 栈内存划分的大小，将直接决定一个JVM进程中可以创建多少个线程（栈内存越大，可创建的线程数量越少，反比）
- 进程的内存大小为：  堆内存+ 线程数量*栈内存



## 守护线程

问题： JVM程序在正常情况下什么时候退出

答案： The java virtual machine exits when the only threads running are all daemon threads

在正常情况下，若JVM中没有一个非守护线程，则JVM的进程会退出。  

异常情况就是使用System.exit()



```java
    /**
     * Marks this thread as either a {@linkplain #isDaemon daemon} thread
     * or a user thread. The Java Virtual Machine exits when the only
     * threads running are all daemon threads.
     *
     * <p> This method must be invoked before the thread is started.
     *
     * @param  on
     *         if {@code true}, marks this thread as a daemon thread
     *
     * @throws  IllegalThreadStateException
     *          if this thread is {@linkplain #isAlive alive}
     *
     * @throws  SecurityException
     *          if {@link #checkAccess} determines that the current
     *          thread cannot modify this thread
     */
    public final void setDaemon(boolean on) {
        checkAccess();
        if (isAlive()) {
            throw new IllegalThreadStateException();
        }
        daemon = on;
    }

  /* Whether or not the thread is a daemon thread. */
    private boolean     daemon = false;
```



- setDaemon 方法只在线程启动之前生效，
- 如果线程已经死亡，那么再设置setDaemon则会抛非法线程状态异常
- 守护线程经常用作与执行一些后台任务，因此也叫 后台线程 
- 当希望关闭某些线程的时候或者退出JVM进程的时候，一些线程能自动关闭， 可以使用守护线程



```java

public class DaemonThread
{
    public static void main(String[] args) throws InterruptedException
    {
        // main 线程开始
        Thread thread = new Thread(() ->
        {
            while (true)
            {
                try
                {
                    Thread.sleep(1);
                } catch (InterruptedException e)
                {
                    e.printStackTrace();
                }
            }
        });

        thread.setDaemon(true);
        thread.start();
        Thread.sleep(2_000L);
        System.out.println("Main thread finished lifecycle.");
    }
}

//如果不设置 thread.setDaemon(true);，则 JVM 一直无法关闭
```



## 总结：

- 详细了解Thread的构造函数，挖掘里面的各类细节，如 threadStatus =0代表 new状态，构造函数调用init函数，尤其是 stacksize  对Thread的影响
- 了解了线程的父子关系， 默认情况下子线程从父线程那里 继承了守护线程、优先级、ThreadGroup等特性
- 守护线程的特性以及使用场景

