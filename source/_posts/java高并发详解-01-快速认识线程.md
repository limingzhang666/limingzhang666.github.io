---
title: java高并发详解-01-快速认识线程
copyright: true
related_posts: true
date: 2021-02-02 23:52:08
tags: 线程
categories: 高并发详解
---

# 快速认识线程

其实蛮早时候就看过这本书，当时对其中的知识也是一知半解，当时的工作并发量也不是很大，对并发的问题也没有太多的考虑。后面慢慢的认识到了高并发的重要性，加上看完了jvm的原理，带着jvm的一些知识点，重读一遍这本书，相信会解开之前的一些疑问。总之，书读百遍，其意自现。每一遍都会得到不一样的收获。
在此记录一下

## 1.1 线程的介绍

线程是程序执行的一个路径，每一个线程都有自己的局部变量表、程序计数器以及各自的生命周期

## 1.2 线程的生命周期

![](/uploads/java-concurrency-master/thread-lifecycle.png)

线程可以大致分为5个阶段

- NEW
- RUNNABLE
- RUNNING
- BLOCKED
- TERMINATED

### 1.2.1 线程的new状态

当我们用关键字new 创建一个Thread 对象时，此时它并不处于执行状态，因为没有调用start方法启动该线程，那么该线程的状态为new状态。

准确的来说，它只是Thread对象的状态，因为再没有start之前，该线程根本不存在。

new状态通过start方法进入Runnable状态

### 1.2.2 线程的RUNNABLE 状态

- 线程对象必须调用start方法 进入RUNNABLE状态，那么此时 才是真正的再JVM进程中创建了一个线程。
- 并不是线程一启动就直接得到执行的，线程的运行与否和进程一样都要听令于CPU的调用。我们把这个中间状态 称为 **可执行状态（RUNNABLE）**
- 也就是说 它具备执行的资格，但是并没有真正的执行起来，而是在等待CPU的调度
- 由于存在Running状态，所以不会直接进入 BLOCKED 状态和 TERMINATED状态，**即使是在线程的执行逻辑中调用wait、sleep或者其他block的 IO操作等，也必须要先获得 CPU的调度执行权才可以**，严格来讲，RUNNABLE的线程只能意外终止或者进入RUNNING状态

### 1.2.3 线程的RUNNING状态

cpu通过轮询或者其他方式从任务可执行队列中选中了线程，那么此时它才能真正地执行自己的逻辑代码，

running状态的线程，可以发生以下的状态转换：

- 直接进入TERMINATED 状态，比如调用jdk已经不推荐使用的stop方法或者判断某个逻辑标识
- 进入BLOCKED状态，比如 调用了sleep 或者wait方法而加入了waitSet中
- 进行某个阻塞的IO操作，比如因网络数据的读写而进入了BLOCKED状态
- 获取某个锁资源，从而加入到该锁的阻塞队列中，从而进入了BOLCKED状态
- 由于CPU的调度器轮询使得该线程放弃执行，进入RUNNABLE状态
- 线程主动调用yield方法，放弃CPU执行权，进入RUNNABLE状态



### 1.2.4 线程的BLOCKED状态

线程在BLOCKED状态中可以切换至如下几个状态：

- 直接进入TERMINATED状态，比如调用JDK已经不推荐使用的stop方法或者意外死亡（jvm  Crash）
- 线程阻塞的操作结束，比如读取了想要的数据字节进入到RUNNABLE状态
- 线程完成了指定时间的休眠，进入到了RUNNABLE状态
- wait中的线程被其他线程 notify /notify all 唤醒，进入runnable状态 
- 线程获取到了某个锁资源，进入到 RUNNABLE 状态
- 线程在阻塞过程中被打断，比如其他线程调用了interrupt方法，进入RUNABLE 状态



### 1.2.5 线程的TERMINATED状态

TERMINATED 是一个线程的最终状态，在该状态中线程将不会切换到其他任何状态，线程进入 TERMINATED 状态，意味着该线程的整个生命周期都结束了，

下面这些情况将会使线程进入 TERMINATED状态。

- 线程运行正常结束，结束生命周期
- 线程运行出错意外结束
- JVM Crash，导致所有 的线程都结束



## 1.3 线程start方法剖析：模板设计模式在Thread中的应用

Thread start方法的源码

```java
 /**
     * Causes this thread to begin execution; the Java Virtual Machine
     * calls the <code>run</code> method of this thread.
     * <p>
     * The result is that two threads are running concurrently: the
     * current thread (which returns from the call to the
     * <code>start</code> method) and the other thread (which executes its
     * <code>run</code> method).
     * <p>
     * It is never legal to start a thread more than once.
     * In particular, a thread may not be restarted once it has completed
     * execution.
     *
     * @exception  IllegalThreadStateException  if the thread was already
     *               started.
     * @see        #run()
     * @see        #stop()
     */
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }

    private native void start0();
```

解释：  start方法的源码非常简单，其实最核心的部分是 start0这个本地方法，也就是JNI方法： 

 private native void start0(); 

也就是说在start方法中 会调用本地方法 start0方法 

通过start方法的注释说明： Causes this thread to begin execution; the Java Virtual Machine   calls the <code>run</code> method of this thread.  

**在开始执行这个线程的时候，JVM将会调用调用该线程的run方法**，换言之，***run方法是被JNI方法start0（）调用的***，。

总结如下几个知识要点：

- Thread被构造后的NEW状态，事实上threadStatus这个内部属性为0 
- 不能两次启动Thread，否则就会出现 IllegalThreadStatusException 异常
- 线程启动后将会被加入到一个ThreadGroup中，
- 一个线程生命周期结束，也就是到了TERMINATED状态，再次调用start方法是不允许的，也就是说 TERMINATED状态时没有办法回到 RUNNABLE/RUNNING 状态的。

### 1.3.2  模板设计模式在Thread中的应用

通过上面分析我们知道，线程的真正执行逻辑时在 run方法中，通常 将run方法称为线程的执行单元，

Thread中run方法的代码如下：

（重写run方法，用start方法启动线程）

如果我们没有使用Runnable 接口对其改造，则可以认为Thread的run方法本身就是一个空的实现

```java
  /**
     * If this thread was constructed using a separate
     * <code>Runnable</code> run object, then that
     * <code>Runnable</code> object's <code>run</code> method is called;
     * otherwise, this method does nothing and returns.
     * <p>
     * Subclasses of <code>Thread</code> should override this method.
     *
     * @see     #start()
     * @see     #stop()
     * @see     #Thread(ThreadGroup, Runnable, String)
     */
    @Override
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

其实 Thread的run 和 start 就是一个比较典型的模板设计模式，父类编写算法结构代码 ，子类实现逻辑细节  ，

```java
public class TemplateMethod {

    public final void print(String message) {
        System.out.println("################");
        wrapPrint(message);
        System.out.println("################");
    }

    protected void wrapPrint(String message) {

    }

    public static void main(String[] args) {
        TemplateMethod t1 = new TemplateMethod(){
            @Override
            protected void wrapPrint(String message) {
                System.out.println("*"+message+"*");
            }
        };
        t1.print("Hello Thread");

        TemplateMethod t2 = new TemplateMethod(){
            @Override
            protected void wrapPrint(String message) {
                System.out.println("+"+message+"+");
            }
        };

        t2.print("Hello Thread");

    }
}
```

print  方法类似于Thread 的start 方法，而 wrapPrint 则类似于 run方法，这样做的好处是： 

- 程序结构由父类控制，并且是final 修饰的，不允许被重写 
- 子类只需要实现想要的逻辑任务即可 

## Runnable 接口的引入以及策略模式在Thread中的使用



### 1.5.1 Runnable的职责 

Runnable 接口只定义了一个无参数无返回值的run方法 ，

```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
```

在很多软文以及一些书籍中，经常会提到：创建线程有两种方式：

- 第一种是构建一个Thread
- 第二种是实现Runnable接口，

这种说法是错误的，最起码是不严谨的，在JDK 种代表线程的只有 Thread这个类。线程的执行单元是 run方法，

- 我们可以通过继承Thread，然后重写run方法实现自己的业务逻辑
- 也可以实现 Runnable 接口实现自己的业务逻辑，代码如下： 

```java
 /**
     * If this thread was constructed using a separate
     * <code>Runnable</code> run object, then that
     * <code>Runnable</code> object's <code>run</code> method is called;
     * otherwise, this method does nothing and returns.
     * <p>
     * Subclasses of <code>Thread</code> should override this method.
     *
     * @see     #start()
     * @see     #stop()
     * @see     #Thread(ThreadGroup, Runnable, String)
     */
    @Override
    public void run() {
        // 如果构造 Thread时 传递了Runnable，则会执行 runnable的run方法
        if (target != null) {
            target.run();
        }
        //否则 需要重写 Thread类的run方法
    }
```

准确的讲， 创建线程只有一种方式，那就是构造Thread类 ，而实现线程的执行单元有2种方式 ： 一种是重写Thread的run方法，另一种 是实现Runnable接口的run方法，并且将Runnable 实例用作构造Thread的参数 。



### 1.5.2 策略模式在Thread 中的应用

无论是 Runnable的run方法，还是Thread类本身的run方法（事实上Thread类也是实现了Runnable接口） 都是想将线程的控制本身 和业务逻辑的运行分离开来，达到职责分明、功能单一的原则，

```java
import java.sql.ResultSet;

public interface RowHandler<T>
{

    T handle(ResultSet rs);
}

```

RowHandler 接口只负责对从数据中查询出来的结果集 进行操作，至于最终返回成什么样的数据结构，那就需要自己去实现，类似于Runable接口

```java
public class RecordQuery
{

    private final Connection connection;

    public RecordQuery(Connection connection)
    {
        this.connection = connection;
    }

    /**
    * 只负责将数据查询 出来 
    */
    public <T> T query(RowHandler<T> handler, String sql, Object... params)
            throws SQLException
    {
        try (PreparedStatement stmt = connection.prepareStatement(sql))
        {
            int index = 1;
            for (Object param : params)
            {
                stmt.setObject(index++, param);
            }

            ResultSet resultSet = stmt.executeQuery();
            // 调用 RowHandler进行数据封装
            return handler.handle(resultSet);
        }
    }

}
```

上面代码的好处是，可以用query方法应对任何数据库的查询，返回结果的不同指挥因为 传入的RowHandler 的不同而不同，同样 RecordQuery只负责数据的获取，而 RowHander则负责数据的加工，职责分明，每个类均功能单一 



- 重写Thread类的run方法和实现Runnable 接口的run方法还有一个很重要的不同 ，那就是 **Thread类的run方法是不能共享的**，
- 也就是说 A线程不能把 B线程的run方法当作自己的执行单元，而使用Runnable接口则 很容易实现这一点，使用同一个RUnnable的实例构造不同的Thread实例。



## 总结：

- 线程的概念
- 如何创建一个线程，并且通过重写Thread的run方法和实现Runnable接口的run方法进而实现线程的执行单元
- 了解模板设计模式 以及 策略设计模式。
- 通过 Thread 以及 Runnable的结合，了解如何实现线程控制和业务执行解耦分离