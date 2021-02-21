---
title: java高并发详解-05-线程间通信
copyright: true
comments: true
related_posts: true
date: 2021-02-21 14:50:38
tags: 线程间通信
categories: 高并发详解
---

- 线程间通信又称为 进程内通信，多个线程实现互斥访问共享资源时 会互相发送信号或等待信号
- 主要是 线程之间 wait，notify，notifyAll ，以及背后的原理内幕

- wait方法必须拥有该对象的monitor ，也就是wait方法必须在同步方法中使用**

# 同步阻塞与异步非阻塞

## 同步阻塞消息处理

- 同步Event 提交，客户端等待时间过长 会陷入阻塞，导致二次提交 Event耗时过长
- 由于客户端提交的Event数量不多，导致系统同时受理业务数量有限，也就是系统的整理的吞吐量不高
- 这种一个线程处理一个Event的方式，会导致出现频繁的创建开启和销毁，从而增加系统额外开销
- 在业务达到峰值的时候，大量的业务处理线程阻塞会导致频繁的CPU切换上下文，从而降低系统性能



## 异步非阻塞消息处理

- 客户端不用等到结果处理结束之后才能返回，从而提高了系统的吞吐量和并发量
- 服务端的线程数量在一个可控的范围之内是不会导致太多的CPU上下文切换，从而带来额外的开销
- 服务端线程可以重复利用，这样可以减少不断创建线程带来的资源浪费



# 单线程间通信

## wait和notify

### wait

- wait 和 notify是 Object中的方法，也就是说 JDK中的每一个类都拥有这2个方法

- 下面是 wait的 3个重载方法

```java
// 1.
public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && timeout == 0)) {
            timeout++;
        }

        wait(timeout);
    }
// 2. 
    public final void wait() throws InterruptedException {
        wait(0);
    }
// 3.
    public final native void wait(long timeout) throws InterruptedException;
```

- wait 方法的3个重载方法都将调用wait（long timeout）这个方法，
- wait（）等价于 wait（0），其中0代表永不超时 
- Object的wait（long timeout）方法会导致当前线程进入阻塞，直到 有其他线程调用了 Object的notify 或者 notifyAll方法才能将其唤醒，或者 阻塞时间到达了 timeout时间而自动唤醒
- **wait方法必须拥有该对象的monitor ，也就是wait方法必须在同步方法中使用**
- **当前线程执行了该对象的wait方法之后，将会放弃对该 monitor的所有权并且进入与该对象关联的waitset中，也就说一旦线程执行了某个object 的wait方法之后，他就会释放对该对象 monitor的所有权，**其他线程也会有机会继续争抢该 monitor的所有权 
- （这个就是 Thread:: join的背后的逻辑，thread.join就是调用了wait 方法，所以父线程会 等到子线程执行完毕，才继续执行）



### notify

```java
 /**
     * Wakes up a single thread that is waiting on this object's
     * monitor. If any threads are waiting on this object, one of them
     * is chosen to be awakened. The choice is arbitrary and occurs at
     * the discretion of the implementation. A thread waits on an object's
     * monitor by calling one of the {@code wait} methods.
     * <p>
     * The awakened thread will not be able to proceed until the current
     * thread relinquishes the lock on this object. The awakened thread will
     * compete in the usual manner with any other threads that might be
     * actively competing to synchronize on this object; for example, the
     * awakened thread enjoys no reliable privilege or disadvantage in being
     * the next thread to lock this object.
     * <p>
     * This method should only be called by a thread that is the owner
     * of this object's monitor. A thread becomes the owner of the
     * object's monitor in one of three ways:
     * <ul>
     * <li>By executing a synchronized instance method of that object.
     * <li>By executing the body of a {@code synchronized} statement
     *     that synchronizes on the object.
     * <li>For objects of type {@code Class,} by executing a
     *     synchronized static method of that class.
     * </ul>
     * <p>
     * Only one thread at a time can own an object's monitor.
     *
     * @throws  IllegalMonitorStateException  if the current thread is not
     *               the owner of this object's monitor.
     * @see        java.lang.Object#notifyAll()
     * @see        java.lang.Object#wait()
     */
    public final native void notify();
```

看一下方法说明注释-

- 唤醒**单个**正在等待执行该对象wait 方法的线程
- 如果有很多线程都在等待，其中之一会被选中唤醒， 选择是任意的
- 被唤醒的线程需要重新获取对该对象所关联 monitor的lock 才能继续执行

- 被唤醒的线程将不能继续，直到当前线程放弃对该对象的锁。被唤醒的线程将以通常的方式与任何其他可能正在主动竞争同步这个对象的线程竞争;例如，被唤醒的线程在成为下一个锁定该对象的线程时没有任何可靠的特权或劣势

###  关于wait 和notify的注意事项

- **wait 是可中断方法**，所以： 当前线程一旦调用了wait方法 进入阻塞状态，其他线程是可以使用 interrupt 方法将其打断的； 可中断方法被打断后，会收到 中断异常 InterruptedException ，同时 interrupt flag 也会被擦除 

- 线程执行了某个对象的wait 方法之后，会加入与之 对应的wait set中，**每一个对象的 monitor 都有一个与之关联的 wait set**
- **当线程进入wait set之后，notify 方法可以将其唤醒**，也就是 从 wait set中弹出，**同时中断 wait 中的线程也会将其唤醒**
- **必须在同步方法中 使用 wait 和 notify 方法，** 因为执行wait 和 notify 的前提条件是  必须持有同步方法的monitor 的所有权， 运行下面任何一个方法 都会抛出 非法的 monitor 状态异常  InllegalMonitorStateException: 

```java
// 报错，因为 **必须在同步方法中 使用 wait 和 notify 方法，**
private void testwait(){
    try{
      this.wait();  
    }catch(InterruptedException e){
        e.printStackTrace();
    }
}

// 报错，**必须在同步方法中 使用 wait 和 notify 方法，**
private void testwait(){
    this.notify();
}
```

- 同步代码的monitor 必须与执行  wait  notify方法的对象一致，简单来说就是 **用哪个对象的 monitor 进行同步，就只能用哪个对象 进行wait 和 notify操作。**

运行下面代码的任何一个方法会报错：

```java
public class WaitNotify
{

    private final Object MUTEX = new Object();

    // 报错，如果非要用 MUTEx ，可以使用同步代码块的形式 
    private synchronized void testWait()
    {
        try
        {
            MUTEX.wait();
        } catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }
    // 报错，报错，如果非要用 MUTEx ，可以使用同步代码块的形式 
    private synchronized void testNotify()
    {
        MUTEX.notifyAll();
    }

    public static void main(String[] args)
    {
        WaitNotify waitNotify = new WaitNotify();
        waitNotify.testNotify();
    }
}

运行结果：

Exception in thread "main" java.lang.IllegalMonitorStateException
	at java.lang.Object.notifyAll(Native Method)
	at com.wangwenjun.concurrent.chapter05.WaitNotify.testNotify(WaitNotify.java:21)
	at com.wangwenjun.concurrent.chapter05.WaitNotify.main(WaitNotify.java:27)

```

上面同步方法中 monitor 引用的是 this，而wait notify 使用的确实  MUTEX，**虽然是在 同步方法中执行 wait  notify方法，但是 wait 和 notify方法的执行并未 获取 MUTEX 的monitor 为前提**





## wait 和sleep

- 都可以使线程进入 阻塞状态
- 都是可中断方法，被中断后会收到中断异常
- wait 时 Object方法， sleep 是 THread  方法
- wait 执行需要在 synchronized 方法中进行，而sleep 不需要
- **线程在同步方法中执行sleep 方法时，不会释放 monitor锁 ， 而 wait 方法则会释放monitor 锁**
- sleep 方法短暂休眠后会主动退出阻塞 ，而 wait方法（没有指定时间） 则需要被其他线程中断才能退出阻塞 





# 多线程通信