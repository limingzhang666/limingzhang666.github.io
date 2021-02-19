---
title: java高并发详解-04-线程安全与数据同步
copyright: true
related_posts: true
date: 2021-02-18 23:56:02
tags: 线程安全与数据同步
categories: 高并发详解
---



- 从字节码指令维度掌握 synchronized 关键字的原理,以及互斥同步的 流程
- 还有一点很重要的是,通过画时序图 分析数据不一致的场景 和原因

# 4.1数据同步(Todo)

### 共享资源: 

多个线程同时对同一份资源进行访问(读写操作),被多个线程访问的资源,**就称为共享资源**

### 数据同步

如何保证多个线程访问到的数据是一致的,则被称为**数据同步或者资源同步**

### 4.1.1 数据不一致问题的引入

```java
public class TicketWindownRunnableError implements Runnable {
    private int index = 1;

    private final static int MAX = 500;

    @Override
    public void run() {
        while (index <= MAX) {
            System.out.println(Thread.currentThread() + "的号码是：" + (index++));
        }
    }

    public static void main(String[] args) {
        final TicketWindownRunnableError task = new TicketWindownRunnableError();
        Thread 一号窗口 = new Thread(task, "一号窗口");
        Thread 二号窗口 = new Thread(task, "二号窗口");
        Thread 三号窗口 = new Thread(task, "三号窗口");
        Thread 四号窗口 = new Thread(task, "四号窗口");
        一号窗口.start();
        二号窗口.start();
        三号窗口.start();
        四号窗口.start();
 
    }
}


```

#### 问题现象

多次运行上面程序，会大致出现3类问题，具体如下：

- 某个号码被略过，没有出现
- 某个号码被多次显示
- 号码超过了最大值500

#### 问题分析



```java
    @Override
    public void run() {
        while (index <= MAX) {
            System.out.println(Thread.currentThread() + "的号码是：" + (index++));
        }
    }
```

##### 1. 号码被掠过

线程的执行是由CPU时间片轮询调度的，假设此时线程1.线程2 都执行到了index =65 的位置，其中线程2 将index 修改为66后未输出前，cpu调度将执行权力交给了线程1，线程1 将其累加到了 67， 那么66就被忽略了

```mermaid
sequenceDiagram
autonumber
note over Thread1,Thread2: Index=65
Thread1 ->> Thread1 :thread 停顿
Thread2 ->> Thread2 : index+1 =65+1
Thread2 ->> Thread2 : index=66
Thread2 ->> Thread2 : thread停顿,问题点，此处忽略了print66，
Thread1 ->> Thread1 :index+1=66+1
Thread1 ->> Thread1 :index=67
Thread1 ->> Thread1 :print 67
Thread1 ->> Thread1 : index+1=67+1
Thread1 ->> Thread1 :index=68
Thread1 ->> Thread1 : print 68


```



##### 2. 号码重复出现

线程1 执行index+1，然后cpu 执行权落入线程2手里，由于线程1 并没有给index 赋值301， 所以线程2 执行index+1的 结果也是 301，

所以出现了 重复号码的情况

```mermaid
sequenceDiagram
autonumber
note over Thread1,Thread2: Index=300
Thread1 ->> Thread1 :index +1=300+1
Thread1 ->> Thread1 : 线程停顿
Thread2 ->> Thread2 : index+1 =300+1
Thread2 ->> Thread2 : index=301
Thread1 ->> Thread1 : index=301
Thread1 ->> Thread1 : print 301
Thread2 ->> Thread2 : 问题点-print 301



```

##### 3. 号码超过了最大值

当 index=499 的时候，线程1 和线程2 都看到条件满足。线程2短暂停顿，线程1 将index增加到了500，线程2恢复运行后，又将 500+1 ，此时就出现了超过MAX 的情况。
```mermaid
sequenceDiagram
autonumber
note over Thread1,Thread2: Index=499
Thread1 ->> Thread1 :进入while 循环条件满足
Thread2 ->> Thread2 :进入while 循环条件满足
Thread1 ->> Thread1 : index+1 =499+1
Thread2 ->> Thread2 : 线程停顿
Thread1 ->> Thread1 : index=500
Thread1 ->> Thread1 : print 500
Thread2 ->> Thread2 : index+1=500+1
Thread2 ->> Thread2 : index =501
Thread2 ->> Thread2 : 问题点-print 501
```

# 4.2 初始synchronized关键字

## 4.2.1 什么是synchronized

JDK官网对synchronized关键字的权威解释：

Synchronized keyword enable a simple strategy for preventing thread interference and memory consistency errors:  if an object is visible to more than one thread,all reads or writes to that object's  variables are done through synchronized methods.

Synchronized关键字启用了一种简单的策略来防止线程干扰和内存一致性错误:如果一个对象对多个线程可见，那么对该对象变量的所有读或写操作都通过Synchronized同步的方法完成。 

同步互斥： 互斥是方式，同步是结果

具体表现为：

- synchronized 关键字提供了一种锁机制，能够确保共享变量的互斥访问，从而防止数据不一致问题的出现
- synchronized 关键字包括 monitor enter 和 monitor exit 两个JVM 指令，它能够保证在任何时候任何线程执行到 monitor enter成功之前都必须从主内存获取数据，而不是从缓存中， 在monitor exit运行成功之后，共享变量被更新后的值 必须刷入回之内存 （简单来说，就是 执行 monitor enter之前，从主内存中获取共享变量的值，在执行 monitor exit之后，将更新后的共享变量的值 同步回主内存，解决缓存一致性的问题）
- synchronized 的指令严格遵守 java 的先行发生原则（happens-before），一个monitor exit 指令之前 一定由一个 monitor enter 指令，成对出现 。



## 4.2.2 synchronized 关键字的用法

synchronized 可以用于对 方法块 或者方法进行修饰， 不能够用于对class 或者变量进行修饰

### 1. 同步方法

同步方法的语法  ： [default | public | private | protected] **synchronized** [static ] type methods().

```java
public synchronized void sync(){
    。。。
}
public synchronized static void staticSync(){
    。。。
}

```

### 2. 同步代码块

```java
private final Object MUTEX=new Object();
public void sync(){
    synchronized(MUTEX){
        。。。
    }
    
}
```

## 4.3 深入理解synchronized关键字

- synchronized 提供了一种互斥机制，在同一时刻，只能由一个线程访问同步资源
- 不应该将MUTEX称为锁，严谨的说法，应该是 某线程 获取到了 MUTEX关联的 monitor锁

### Monitor enter

每个对象都与一个monitor 相关联，一个monitor的lock的锁 只能被一个线程在同一时间获得，在一个线程尝试获得与 对象关联monitor的所有权时候回发生如下几件事情。

- 如果monitor 的计数器为0，则意味着 该monitor的lock 还没有被获得，某个线程获得之后 将立即对该计数器加一 ，从此该线程就是这个monitor 的所有者了
- 如果一个已经拥有该monitor所有权的线程重入 ，则会导致 monitor计数器再次累加
- 如果monitor已经被其他线程 所拥有，则 其他线程尝试获取该monitor 的所有权时，会被陷入阻塞状态知道monitor计数器 变为0，才能再次尝试 获取对 monitor的所有权 。

### Monitor exit

- 释放对 monitor的所有权，想要释放对 某个对象关联的 monitor的所有权的前提是 ，你曾经获得了所有权。
- 释放monitor锁的过程比较简单，就是将 monitor的计数器减一， 
- 如果monitor的计数器 结果为0，那就意味着 该线程不再拥有对 该monitor的所有权，通俗的讲 就是解锁。
- 与此同时，被该 monitor block的线程将再次尝试 获得对该monitor的所有权

### 使用 synchronized需要注意的问题

- 与monitor关联的对象不能为空,  

```java
private final Object mutex=null;
public void syncMethod(){
    synchronized(mutex){
        、、、
    }
}
每一个对象和一个 monitor 关联，对象都为null了，monitor 肯定无从谈起
```

- synchronized作用域太大，synchronized 应该尽可能地只作用于  **共享资源地读写作用域**

```java
public static class Task implements Runnable{
    @Override
    public synchronized void run(){
        //
    } 
}
由于synchronized 关键字存在排他性，也就是所有线程必须串行的经过synchronized保护的共享区域，
    如果synchronized 作用域越大，则代表其效率越低，甚至丧失并发的优势
```

- 不同的monitor 企图锁相同的方法

  ```java
  public static class Task implements Runnable{
      private final Object mutex=new Object();
      @Override
      public synchronized void run(){
          //
           synchronized(mutex){
          、、、
     	 }
      } 
      
      public static void main(String[] args){
          for (int i=0;i<5;i++){
              new Thread(Task::new).start();
          }
      }
      
  }
  //构造了5个线程，同时也构造了5个 Runnable实例， Runnable作为逻辑单元传递给Thread，然后将发现：
  synchronized 无法同步互斥， 因为 线程之间的monitor lock争抢只能发生在 monitor 关联的同一个引用上。
  ```



- 多个锁的交叉导致死锁



### This Monitor和Class Monitor



TODO： 

This Monitor:

实例锁



Class Monitor:

对象锁，static修饰