---
title: java高并发详解-03-ThreadAPI详解
copyright: true
related_posts: true
date: 2021-02-18 13:25:17
tags: ThreadAPI
categories: 高并发详解
---

## 线程sleep

sleep是一个静态方法，其中有两个重载方法，其中一个需要传入毫秒数，另外一个既需要毫秒数也需要纳秒数

```java
/**
     * Causes the currently executing thread to sleep (temporarily cease
     * execution) for the specified number of milliseconds, subject to
     * the precision and accuracy of system timers and schedulers. The thread
     * does not lose ownership of any monitors.
     *
     * @param  millis
     *         the length of time to sleep in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public static native void sleep(long millis) throws InterruptedException;

    /**
     * Causes the currently executing thread to sleep (temporarily cease
     * execution) for the specified number of milliseconds plus the specified
     * number of nanoseconds, subject to the precision and accuracy of system
     * timers and schedulers. The thread does not lose ownership of any
     * monitors.
     *
     * @param  millis
     *         the length of time to sleep in milliseconds
     *
     * @param  nanos
     *         {@code 0-999999} additional nanoseconds to sleep
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative, or the value of
     *          {@code nanos} is not in the range {@code 0-999999}
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public static void sleep(long millis, int nanos) throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        sleep(millis);
    }
```

- **sleep方法会使得当前线程进入指定毫秒数的休眠，暂停执行，虽然给定一个休眠时间，但是最终要以系统的定时器和调度器的精度为准。**

- **sleep 期间不会放弃 monitor 锁的所有权** 

  

### 使用TimeUnit 替代 Thread.sleep

- 在JDK引入了一个枚举 TimeUnit，其对 sleep 方法提供了很好的封装。
- 强烈建议，使用TimeUnit代替 直接使用Thread.sleep()



## 线程yield

- Thread.yeild 线程礼让，当前线程暂时不跑了，让其他线程先跑。类似于你去银行排队办事情，你跑到最后去重新拿个号重新排队。

```java
  /**
     * A hint to the scheduler that the current thread is willing to yield
     * its current use of a processor. The scheduler is free to ignore this
     * hint.
     * // 意思是，给了调度器scheduler一个提示，我愿意让出当前的处理器processor给其他人，但是人家processor未必搭理你这个暗示。
     * <p> Yield is a heuristic attempt to improve relative progression
     * between threads that would otherwise over-utilise a CPU. Its use
     * should be combined with detailed profiling and benchmarking to
     * ensure that it actually has the desired effect.
     *
     这个例子就是通过yield方法来实现两个线程的交替执行。
   		不过请注意：这种交替并不一定能得到保证，源码中也对这个问题进行说明：
   		主要说明了三个问题：
   　　调度器可能会忽略该方法。
   　　使用的时候要仔细分析和测试，确保能达到预期的效果。
   　　很少有场景要用到该方法，主要使用的地方是调试和测试。　　
     * <p> It is rarely appropriate to use this method. It may be useful
     * for debugging or testing purposes, where it may help to reproduce
     * bugs due to race conditions. It may also be useful when designing
     * concurrency control constructs such as the ones in the
     * {@link java.util.concurrent.locks} package.
     */
    public static native void yield();
```



- yield方法属于一种**启发式的方法**，会提醒调度器 我愿意放弃当前CPU资源，如果CPU资源不紧张，则会忽略这种提醒

- 调用 yield 方法 会使当前线程从 ＲＵＮＮＩＮＧ　状态切换到　ＲＵＮＮＡＢＬＥ

- yield 只是一个提示(hint),cpu 调度器并不会担保每次都能满足 yield提示.

  ### yield 和 sleep

  - sleep 会导致当前线程暂停指定的时间,没有CPU时间片的消耗
  - yield 只是对CPU调度器的一个提示,如果CPU调度器没有忽略这个提示, 他会导致线程上下文的切换.(因为当前线程愿意让出自己的资源)
  - sleep会使 线程短暂block ,会在给定的时间内释放 CPU资源 
  - yield 会使 RUNNING状态的Ｔｈｒｅａｄ　进入　ＲＵＮＮＡＢＬＥ状态(如果CPU调度器没有忽略这个提示的话)
  - sleep几乎百分百地完成了给定时间的休眠,而 yield 的提示并不能一定担保
  - 一个线程sleep 另一个线程调用interrupt 会捕获到中断信号,而 yield 则不会
  - **yield 方法和同步没关系，也就是和ObjectMonitor没关系，你硬上锁就是在唱独角戏 ( _05_03_YieldTest）**

## 设置线程的优先级

```java
   /**
     * Changes the priority of this thread.
     * <p>
     * First the <code>checkAccess</code> method of this thread is called
     * with no arguments. This may result in throwing a
     * <code>SecurityException</code>.
     * <p>
     * Otherwise, the priority of this thread is set to the smaller of
     * the specified <code>newPriority</code> and the maximum permitted
     * priority of the thread's thread group.
     */
    public final void setPriority(int newPriority) {
        ThreadGroup g;
        checkAccess();
        if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
            throw new IllegalArgumentException();
        }
        if((g = getThreadGroup()) != null) {
            if (newPriority > g.getMaxPriority()) {
                newPriority = g.getMaxPriority();
            }
            setPriority0(priority = newPriority);
        }
    }

    /**
     * Returns this thread's priority.
     *
     * @return  this thread's priority.
     * @see     #setPriority
     */
    public final int getPriority() {
        return priority;
    }
```

- 理论上优先级高的线程会优先获取到  被cpu调度的机会,但是**这个优先级 和yield一样,同样只是一个  hint(提示)**
- 对于root用户,他会 hint  系统你想要设置的优先级别, 否则他会被忽略
- 如果CPU比较忙,设置优先级可能会获得更多的CPU时间片,但是 在闲时优先级的高低几乎不会有任何作用
- **不要在程序设计当中企图使用线程优先级绑定某些特定的业务,或者让业务严重依赖线程优先级**

- 线程优先级 区间范围为 [1,10],如果不在该区间 则 抛出异常
- 如果set设置的优先级大于 ThreadGroup的优先级, 则以 ThreadGroup为准 .
- 线程默认的优先级与父类保持一致, 一般情况下是 5,因为main线程的优先级就是5,所以它派生出来的线程都是5.



## 获取线程唯一ID

```java
  /**
     * Returns the identifier of this Thread.  The thread ID is a positive
     * <tt>long</tt> number generated when this thread was created.
     * The thread ID is unique and remains unchanged during its lifetime.
     * When a thread is terminated, this thread ID may be reused.
     *
     * @return this thread's ID.
     * @since 1.5
     */
    public long getId() {
        return tid;
    }
```

- 线程的ID在整个JVM进程中都会是唯一的,并且是从 0开始逐次递增.
- 如果在main方法(main线程)中创建了一个唯一的线程,并且调用getid方法 后发现返回结果并不等于0 ,不必惊讶,因为一个JVM启动时候,实际上已经开辟了很多个线程.自增序列已经有所增加了,所以我们创建的并非是第0号线程

## 获取当前线程

```java
  /**
     * Returns a reference to the currently executing thread object.
     *
     * @return  the currently executing thread.
     */
    public static native Thread currentThread();
```

- currentThread() 用于返回当前执行线程的引用,这个方法虽然很简单,但是使用非常广泛,

## 设置线程上下文类加载器



```java

    /* The context ClassLoader for this thread */
    private ClassLoader contextClassLoader;   

public void setContextClassLoader(ClassLoader cl) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("setContextClassLoader"));
        }
        contextClassLoader = cl;
    }

    @CallerSensitive
    public ClassLoader getContextClassLoader() {
        if (contextClassLoader == null)
            return null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            ClassLoader.checkClassLoaderPermission(contextClassLoader,
                                                   Reflection.getCallerClass());
        }
        return contextClassLoader;
    }

```

- getContextClassLoader ()
	- 获取线程上下文的类加载器, 简单来说就是这个线程是由哪个类加载器加载的.
	- 如果在没有修改线程上下文类加载器的情况下,则保持与父类同样的类加载器
- setContextClassLoader(ClassLoader cl) 设置该线程的类加载器,
  - 该方法可以打破java类加载的双亲加载(父委托机制),有时候也称为 **Java类加载器的后门**
  - 后续会有专门的讲解



## 线程interrupt

**中断线程阻塞**

**这是一个 非常重要的API**,也是经常使用的方法,与线程中断的API有如下几个,我们通过源码详解解析

- public void interrupt() 
- public static boolean interrupted()
- public boolean isInterrupted()



### interrupt() 

```java
 /*
     * Interrupts this thread.(打断这个线程)   
     * <p> Unless the current thread is interrupting itself, which is
     * always permitted, the {@link #checkAccess() checkAccess} method
     * of this thread is invoked, which may cause a {@link
     * SecurityException} to be thrown.
     *
     * <p> If this thread is blocked in an invocation of the {@link
     * Object#wait() wait()}, {@link Object#wait(long) wait(long)}, or {@link
     * Object#wait(long, int) wait(long, int)} methods of the {@link Object}
     * class, or of the {@link #join()}, {@link #join(long)}, {@link
     * #join(long, int)}, {@link #sleep(long)}, or {@link #sleep(long, int)},
     * methods of this class, then its interrupt status will be cleared and it
     * will receive an {@link InterruptedException}.
     // 如果当前线程被阻塞了由于调用下面这些什么Object的wait或者Thread的 sleep .join方法,
     然后它的中断状态将被清除 会收到{@link InterruptedException}。
     *
     * <p> If this thread is blocked in an I/O operation upon an {@link
     * java.nio.channels.InterruptibleChannel InterruptibleChannel}
     * then the channel will be closed, the thread's interrupt
     * status will be set, and the thread will receive a {@link
     * java.nio.channels.ClosedByInterruptException}.
     *
     * <p> If this thread is blocked in a {@link java.nio.channels.Selector}
     * then the thread's interrupt status will be set and it will return
     * immediately from the selection operation, possibly with a non-zero
     * value, just as if the selector's {@link
     * java.nio.channels.Selector#wakeup wakeup} method were invoked.
     *
     * <p> If none of the previous conditions hold then this thread's interrupt
     * status will be set. </p>
     *
     * <p> Interrupting a thread that is not alive need not have any effect.
         */
public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }

   private native void interrupt0();
```

- 如果 当前线程正处于 阻塞状态,调用 interrupt方法,则可以 **中断打断这个阻塞**

- 是线程进入阻塞的方法有:

  - Object的 wait 方法 以及变形的 重载的方法:wait (long)    和wait (long,int)

  - Thread 的sleep(long)方法,以及 重载的方法

  - Thread的join方法 以及重载的方法

  - InterruptibleChannel 的io操作

  - Selector 的wakeup方法

  - 其他方法

    - 以上的方法都会使得当前线程进入阻塞状态.如果另外一个线程调用被阻塞线程的 interrupt 方法,则会打破这种阻塞 .
    - 打断一个线程并不等于该线程的生命周期结束,仅仅是 打断了当前线程的阻塞状态
    - 一个线程在阻塞的情况下被打断,会抛出一个 InterruptedException 的异常,这个异常就像一个 signal 一样通知当前线程被打断了 

    
### interrupt源码解析 

- 一个线程内存存在着名为 interrupt flag的标识,如果一个 线程被interrupt ,那么它的 flag 将被设置
- 通过源码可以看到Thread中存在一个私有方法: **interrupt0();           // Just to set the interrupt flag**",该方法作用是  修改interrupt flag
- 如果一个线程已经是 死亡Terminated 状态,那么尝试对其的interrupt 会直接被忽略\




### isInterrupted

```java
 /**
     * Tests whether this thread has been interrupted.  The <i>interrupted
     * status</i> of the thread is unaffected by this method.
     *
     * <p>A thread interruption ignored because a thread was not alive
     * at the time of the interrupt will be reflected by this method
     * returning false.
     *
     * @return  <code>true</code> if this thread has been interrupted;
     *          <code>false</code> otherwise.
     * @see     #interrupted()
     * @revised 6.0
     */
    public boolean isInterrupted() {
        return isInterrupted(false);
    }

   /**
     * Tests if some Thread has been interrupted.  The interrupted state
     * is reset or not based on the value of ClearInterrupted that is
     * passed.
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```

- 主要判断当前线程是否被中断,该方法仅仅是对 interrupt flag 的一个判断,并不会影响改变 interrupt flag的值
- 可中断方法捕获到了中断信号(signal) 之后,也就是捕获了InterruptedException 异常之后,会擦除interrupt的标识.
- 可中断方法捕获到了中断信号后,为了不影响线程中的其他方法的执行,将线程的interrupt flag标识复位 ,很合理的 设计




### interrupted

```java
 /**
     * Tests whether the current thread has been interrupted.  The
     * <i>interrupted status</i> of the thread is cleared by this method.  In
     * other words, if this method were to be called twice in succession, the
     * second call would return false (unless the current thread were
     * interrupted again, after the first call had cleared its interrupted
     * status and before the second call had examined it).
     *
     测试当前线程是否已被中断的 . 这个线程的 <中断状态>被这个方法清除。
	*换句话说，如果这个方法被连续调用两次，则第二个调用将返回false(
    除非当前线程被再一次 interrupted,在第一次调用后 已经清除了它自身 的interrupted status后再次被中断状态和第二次调用之前检查它
 
     
     * <p>A thread interruption ignored because a thread was not alive
     * at the time of the interrupt will be reflected by this method
     * returning false.
     线程中断被忽略，因为线程不是活的中断的时间会被这个方法反映出来*返回false。
     *
     * @return  <code>true</code> if the current thread has been interrupted;
     *          <code>false</code> otherwise.
     * @see #isInterrupted()
     * @revised 6.0
     */
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }


   /**
     * Tests if some Thread has been interrupted.  The interrupted state
     * is reset or not based on the value of ClearInterrupted that is
     * passed.
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```

- 调用该方法会直接擦除掉线程的interrupt flag,

- 需要注意的是: 第一次调用interrupted方法会返回true ,并且立即擦除了interrupt flag;

- 第二次包括以后的调用永远都是返回false,除非在此期间又一次地被打断了 .

### interrupted源码分析

- isInterrupted() 方法和   interrupted()方法都调用了同一个 native方法 :isInterrupted(boolean ClearInterrupted);,                                                                 ClearInterrupted用来控制是否擦除线程的 interrupt flag
- isInterrupted()的 参数为 false,表示 不想擦除
- interrupt 静态方法中该参数为 true,表示想擦除 



如果一个线程在没有执行可中断方法之前就被打断了,那么其接下来执行可中断方法,比如sleep 会发生什么情况呢:

```java
public class ThreadInterrupt
{
    public static void main(String[] args)
    {

    	//  1.判断当前线程是否被中断
        System.out.println("Main thread is interrupted? " + Thread.interrupted());
        // 2.中断当前线程
        Thread.currentThread().interrupt();
        //  3.判断当前线程是否被中断
        System.out.println("Main thread is interrupted? " + Thread.currentThread().isInterrupted());
        try
        {
            //4. 当前线程执行可中断方法
            TimeUnit.MINUTES.sleep(1);
        } catch (InterruptedException e)
        {
            // 5.捕获中断信号
            System.out.println("I will be interrupted still.");
        }
    }
}

结果: 
Main thread is interrupted? false
Main thread is interrupted? true
I will be interrupted still.

```

说明: 如果一个线程设置了 interrupt flag,那么接下来可中断方法 会立即中断,因此 注释5的信号捕获部分会被执行.



## 线程Join

```java
    /**
     * Waits for this thread to die.
     *
     * <p> An invocation of this method behaves in exactly the same
     * way as the invocation
     *
     * <blockquote>
     * {@linkplain #join(long) join}{@code (0)}
     * </blockquote>
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public final void join() throws InterruptedException {
        join(0);
    }

    /**
     * Waits at most {@code millis} milliseconds for this thread to
     * die. A timeout of {@code 0} means to wait forever.
     *
     * <p> This implementation uses a loop of {@code this.wait} calls
     * conditioned on {@code this.isAlive}. As a thread terminates the
     * {@code this.notifyAll} method is invoked. It is recommended that
     * applications not use {@code wait}, {@code notify}, or
     * {@code notifyAll} on {@code Thread} instances.
     *
     * @param  millis
     *         the time to wait in milliseconds
     *
     * @throws  IllegalArgumentException
     *          if the value of {@code millis} is negative
     *
     * @throws  InterruptedException
     *          if any thread has interrupted the current thread. The
     *          <i>interrupted status</i> of the current thread is
     *          cleared when this exception is thrown.
     */
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```



join某个线程A(),会使当前线程B进入等待,直到线程A结束生命周期,或者到达给定的时间,

那么在此期间B线程是处于BLOCKED的,而不是A线程,

- join方法会使得当前线程永远的等待下去,知道期间被另外的线程中断,或者join的线程执行结束.
- join的另外2个重载方法,指定毫秒数,在指定的时间到达之后,当前线程也会退出阻塞.

#### 问题: 如果一个线程已经结束了生命周期,那么调用它的join方法的当前线程会被阻塞吗?

```java
  Thread thread = new Thread();
        thread.start();

        TimeUnit.MILLISECONDS.sleep(2);
        thread.join();
        System.out.println("===============");

不会被阻塞
```

-  join() 和 sleep() 一样，可以被中断（被中断时，会抛出 InterrupptedException 异常）；

- 不同的是，join() 内部调用了 wait()，会出让锁，

- 而 sleep() 会一直保持锁。



### join() 的示例和作用

```java
// 父线程
public class Parent {
    public static void main(String[] args) {
        // 创建child对象，此时child表示的线程处于NEW状态
        Child child = new Child();
        // child表示的线程转换为RUNNABLE状态
        child.start();
        // 等待child线程运行完再继续运行
        child.join();
    }
}


// 子线程
public class Child extends Thread {
    public void run() {
        // ...
    }
}
```

上面代码展示了两个类：Parent（父线程类），Child（子线程类）。

Parent.main()方法是程序的入口，通过 Child child = new Child(); 新建child子线程（此时 child子线程处于NEW状态）；

然后调用child.start()（child子线程状态转换为RUNNABLE）；

再调用child.join()，此时，Parent父线程会等待child子线程运行完再继续运行。

下图是我总结的 Java 线程状态转换图：

![](/uploads/java-concurrency-master/thread-join.png)

### join() 的作用

让父线程等待子线程结束之后才能继续运行

Waiting for the finalization of a thread

In some situations, we will have to wait for the finalization of a thread. For example, we may have a program that will begin initializing the resources it needs before proceeding with the rest of the execution. We can run the initialization tasks as threads and wait for its finalization before continuing with the rest of the program. For this purpose, we can use the join() method of the Thread class.   ***When we call this method using a thread object, it suspends the execution of the calling thread until the object called finishes its execution.***


 当我们调用某个线程的这个方法时，这个方法会挂起调用线程，直到被调用线程结束执行，调用线程才会继续执行。

### join源码分析:

join() 一共有三个重载版本，分别是无参、一个参数、两个参数：

```bash
 public final void join() throws InterruptedException;
 public final synchronized void join(long millis) throws InterruptedException;

 public final synchronized void join(long millis, int nanos) throws InterruptedException;
```



其中

(1) 三个方法都被final修饰，无法被子类重写。

(2) join(long), join(long, long) 是synchronized method，同步的对象是当前线程实例。

(2) 无参版本和两个参数版本最终都调用了一个参数的版本。

(3) join() 和 join(0) 是等价的，表示一直等下去；join(非0)表示等待一段时间。

**从源码可以看到 join(0) 调用了Object.wait(0)，其中Object.wait(0) 会一直等待，直到被notify/中断才返回。**

**while(isAlive())是为了防止子线程伪唤醒(spurious wakeup)，只要子线程没有TERMINATED的，父线程就需要继续等下去。**

(4) join() 和 sleep() 一样，可以被中断（被中断时，会抛出 InterrupptedException 异常）；不同的是，join() 内部调用了 wait()，会出让锁，而 sleep() 会一直保持锁。

**以本文开头的代码为例，我们分析一下代码逻辑：**

- 调用链：Parent.main() -> child.join() -> child.join(0) -> child.wait(0)（此时 Parent线程会获得 child 实例作为锁，其他线程可以进入 child.join() ，但不可以进入 child.join(0)， 因为child.join(0)是同步方法）。

- 如果 child 线程是 Active，则调用 child.wait(0)（为了防止子线程 spurious wakeup, 需要将 wait(0) 放入 while(isAlive()) 循环中。

- 一旦 child 线程不为 Active （状态为 TERMINATED）, child.notifyAll() 会被调用-> child.wait(0)返回 -> child.join(0)返回 -> child.join()返回 -> Parent.main()继续执行, 子线程会调用this.notify()，child.wait(0)会返回到child.join(0) ，child.join(0)会返回到 child.join(), child.join() 会返回到 Parent 父线程，Parent 父线程就可以继续运行下去了。

 

- 子线程结束后，子线程的this.notifyAll()会被调用，join()返回，父线程只要获取到锁和CPU，就可以继续运行下去了
- 在调用 join() 方法的程序中，原来的多个线程仍然多个线程，**并没有发生“合并为一个单线程”**。真正发生的是调用 join() 的线程进入 TIMED_WAITING 状态，等待 join() 所属线程运行结束后再继续运行。



## 如何关闭一个线程

### 正常关闭

1. 线程结束生命周期,正常结束

2. 捕获中断信号 关闭线程

   - 通过new Thread的方式创建线程,这种方式看似简单,但是其实 派生成本是比较高的,因此在一个线程中往往会循环地执行某个任务,比如心跳检查,不断接收网络消息报文,系统决定退出地时候,可以借助中断线程地方式使其退出,例如:

     ```java
       public static void main(String[] args) throws InterruptedException
         {
     
             Thread t = new Thread()
             {
                 @Override
                 public void run()
                 {
                     System.out.println("I will start work");
                     for (; ; )
                     {
                         //working.
                         try
                         {
                             TimeUnit.MILLISECONDS.sleep(1);
                         } catch (InterruptedException e)
                         {
                             break;
                         }
                     }
                     System.out.println("I will be exiting.");
                 }
             };
     
             t.start();
             TimeUnit.MINUTES.sleep(1);
             System.out.println("System will be shutdown.");
             t.interrupt();
         }
     
     线程中执行某个可中断方法,可以通过捕获中断信号来决定是否退出
     ```

     3. 使用volatile开关控制

        由于线程的interrupt 标识 很有可能被擦除,或者逻辑单元不会调用任何可中断方法,

        所以使用volatile修饰的开关 flag关闭线程也是一种常见做法

        ```java
        public class FlagThreadExit
        {
            static class MyTask extends Thread
            {
                private volatile boolean closed = false;
                @Override
                public void run()
                {
                    System.out.println("I will start work");
                    while (!closed && !isInterrupted())
                    {
                        //System.out.println("i am working.");
                    }
                    System.out.println("I will be exiting.");
                }
                public void close()
                {
                    this.closed = true;
                    this.interrupt();
                }
            }
            public static void main(String[] args) throws InterruptedException
            {
                MyTask t = new MyTask();
                t.start();
                TimeUnit.SECONDS.sleep(1);
                System.out.println("System will be shutdown.");
                t.close();
            }
        }
        ```

### 异常退出

在一个线程的执行单元中,是不允许抛出 checked异常的, 如果线程在运行过程中,需要捕获checked 异常并且判断是否继续运行,

那么此时可以将checked异常封装成unchecked异常(RuntimeException) 抛出,进而 结束线程的生命周期



### 系统假死

绝大部分原因是因为某个线程阻塞了,或者线程出现了死锁 .



## 总结

- 学习了Thread 的大多数API,主要分为2类,
- 一类是 获取线程的信息,如 getID,getName,getPriority,currThread
- 一类是阻塞以及中断阻塞 方法, sleep,join,  interrupt 