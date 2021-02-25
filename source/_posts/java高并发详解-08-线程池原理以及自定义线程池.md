---
title: java高并发详解-08-线程池原理以及自定义线程池
copyright: true
comments: true
related_posts: true
date: 2021-02-24 20:33:31
tags: 线程池
categories: 高并发详解
---

- 线程池主要是为了重复利用线程，提高效率
- 因为线程是一个重量级的资源，创建、启动以及销毁都是比较耗费资源的，因此对线程重复利用是一种好的程序设计习惯
- 系统中可创建的线程数量是有限的，线程数量和系统性能是一种抛物线的关系
- 本章主要从原理入手，设计一个线程池，理解一个线程池应该具有哪些功能，需要注意哪些细节



# 线程池原理

一个完整的线程池应该具备如下要素

- 任务队列： 用户缓存提交的任务

- 线程数量管理功能： 一个线程池必须能够很好的管理 和控制线程的数量，可以通过3个参数来实现：

  - 创建线程池时 初始的线程数量init
  - 线程池自动扩充的最大线程数量max
  - 在线程池空闲的时候需要释放线程，但是也要维护一定数量的活跃数量或者核心数量 core

  有了这3个参数，就能够很好的控制线程池中的线程数量，将其维护在一个合理的范围之内，三者关系是  init<=core<=max

- 任务拒绝策略： 如果线程数量已达到上限且任务队列已满，则需要有相应的拒绝策略来通知任务提交者。

- 线程工厂： 用于个性化定制线程，比如将线程设置为守护线程以及设置线程名称等

- QueueSize： 任务队列主要存放 提交的Runnable，但是为了防止内存溢出，需要有limit 数量对其进行控制

- KeepedAlive时间： 该时间主要决定线程各个重要参数自动维护的时间间隔

![](/uploads/java-concurrency-master/ThreadPoolPrinciple.png)



# 线程池实现

![](/uploads/java-concurrency-master/ThreadPoolImpl.png)



## 线程池接口定义

1. ThreadPool

   ThreadPool主要顶一个了一个线程池应该具备的基本操作和方法，

```java
public interface ThreadPool {

    // 提交任务到线程池
    void execute(Runnable runnable);
    //关闭线程池
    void shutdown();
    //获取线程池大小,返回初始线程数量
    int getInitSize();
    //获取线程池最大线程数
    int getMaxSize();
    //获取线程池的核心线程数量
    int getCoreSize();
    //获取线程池中用户缓存任务队列的大小
    int getQueueSize();
    //获取线程池中活跃线程的数量
    int getActiveCount();
    //查看线程池是否已经被shutdown
    boolean isShutdown();
}
```

2. RunnableQueue

   RunnableQueue 用于存放提交的Runnable，该Runnable是一个BlockedQueue，并且有limit的限制

   ```java
   /**
    * 任务队列，用户缓存提交到线程池中的任务
    */
   public interface RunnableQueue {
       /**
        * 当有新任务进来时候，首先会offer 到队列中
        *
        * @param runnable
        */
       void offer(Runnable runnable);
   
       /**
        * 工作线程通过take方法 获取Runnable
        *
        * @return
        * @throws InterruptedException
        */
       Runnable take() throws InterruptedException;
   
       /**
        * 获取任务队列中任务的数量
        *
        * @return
        */
       int size();
   }
   ```

   3. ThreadFactory

      THreadFactory 提供创建线程的接口，以便于个性化的定制 Thread，比如应该被加入到哪个 group中，优先级，线程名字以及是否为守护线程等

```java
/**
 * 创建线程的工厂
 */
@FunctionalInterface
public interface ThreadFactory {
    Thread createThread(Runnable runnable);
}
```

4. DenyPolicy

   DenyPolicy主要用于 当Queue中的runnable 达到了limit上限的时候， 决定采用何种策略通知提交者。该接口中默认定义了3中实现。

   ```java
   @FunctionalInterface
   public interface DenyPolicy {
   
       void reject(Runnable runnable, ThreadPool threadPool);
   
       /**
        * 默认实现1： 直接将任务丢弃
        */
       class DiscardDenyPolicy implements DenyPolicy {
   
           @Override
           public void reject(Runnable runnable, ThreadPool threadPool) {
               //do nothing
           }
       }
       /**
        * 默认实现2： 像任务提交者抛出异常
        */
       class AbortDenyPolicy implements DenyPolicy {
   
           @Override
           public void reject(Runnable runnable, ThreadPool threadPool) {
               throw new RunnableDenyException("The runnable " + runnable + " will be abort.");
           }
       }
       /**
        * 默认实现3： 在提交者所在的线程中执行任务
        */
       class RunnerDenyPolicy implements DenyPolicy {
   
           @Override
           public void reject(Runnable runnable, ThreadPool threadPool) {
               if (!threadPool.isShutdown()) {
                   runnable.run();
               }
           }
       }
   }
   ```

   5. RunnableDenyException

      RunnableDenyException 是RuntimeException的子类，主要用于通知 任务提交者，任务队列已无法再接受新的任务

```java
public class RunnableDenyException extends RuntimeException {

    public RunnableDenyException(String message) {
        super(message);
    }
}
```

6. InternalTask

   InternalTask 是 Runnable的一个实现，主要用于线程池内部，该类会使用到 RunnableQueue，然后不断地从queue中取出某个runnable ，并且运行runnable的 run方法

```java
public class InternalTask implements Runnable {

    private final RunnableQueue runnableQueue;

    private volatile boolean running = true;

    public InternalTask(RunnableQueue runnableQueue) {
        this.runnableQueue = runnableQueue;
    }

    @Override
    public void run() {
        // 如果当前任务为 running ，并且没有被中断，
        while (running && !Thread.currentThread().isInterrupted()) {
            try {
                // 则其将不断地 从queue中获取 runnable，然后执行run方法
                Runnable task = runnableQueue.take();
                task.run();
            } catch (InterruptedException e) {
                running = false;
                break;
            }
        }
    }

    //停止当前任务，主要会在 线程池的shutdown 方法中使用
    public void stop() {
        this.running = false;
    }
}
```



## 线程池的详细实现

1. LinkedRunnableQueue (将runnableList 作为同步锁 对象)

   ```java
   public class LinkedRunnableQueue implements RunnableQueue {
   
       // 任务队列的最大容量，在构造时候传入
       private final int limit;
       // 假如任务队列中的任务满了，则需要执行拒绝策略
       private final DenyPolicy denyPolicy;
       // 用于存放任务的队列（双向循环列表）
       private final LinkedList<Runnable> runnableList = new LinkedList<>();
   
       private final ThreadPool threadPool;
   
       public LinkedRunnableQueue(int limit, DenyPolicy denyPolicy, ThreadPool threadPool) {
           this.limit = limit;
           this.denyPolicy = denyPolicy;
           this.threadPool = threadPool;
       }
   
   
       @Override
       public void offer(Runnable runnable) {
           synchronized (runnableList) {
               if (runnableList.size() >= limit) {
                   // 无法容纳新的任务时，执行拒绝策略
                   denyPolicy.reject(runnable, threadPool);
               } else {
                   // 将任务假如到队尾，并且唤醒阻塞中的线程
                   runnableList.addLast(runnable);
                   runnableList.notifyAll();
               }
           }
       }
   
       @Override
       public Runnable take() throws InterruptedException {
           synchronized (runnableList) {
               // 如果任务队列中没有可执行的任务，则将当前线程挂起，进入 runnablelist 关联的wait set中等待唤醒
               // （有新任务假如时，会被唤醒）
               while (runnableList.isEmpty()) {
                   try {
                       runnableList.wait();
                   } catch (InterruptedException e) {
                       // 被中断时，需要将该异常抛出，通知上游 InternalTask
                       throw e;
                   }
               }
               // 从任务队列头部移除一个任务
               return runnableList.removeFirst();
           }
       }
   
   
       @Override
       public int size() {
           //返回当前任务队列中的任务数量
           synchronized (runnableList) {
               return runnableList.size();
           }
       }
   }
   
   ```

   

2.初始化线程池 (线程池本身也是一个线程)

线程池需要有数量控制属性（）、创建线程工厂（ThreadFactory）、任务队列策略（DenyPolicy） 等功能

```java
/**
 * 线程池本身自己也是一个线程，需要keepAlive，更新容量
 */
public class BasicThreadPool extends Thread implements ThreadPool {
    //初始化线程数量
    private final int initSize;
    //线程池最大线程数量
    private final int maxSize;
    //线程池核心线程数量
    private final int coreSize;
    // 当前活跃的线程数量
    private int activeCount;
    // 创建线程工厂
    private final ThreadFactory threadFactory;
    // 任务队列
    private final RunnableQueue runnableQueue;
    // 线程池是否已经被shutdown
    private volatile boolean isShutdown = false;
    // 工作线程队列（存放活跃线程）
    private final Queue<ThreadTask> threadQueue = new ArrayDeque<>();

    private final static DenyPolicy DEFAULT_DENY_POLICY = new DenyPolicy.DiscardDenyPolicy();

    private final static ThreadFactory DEFAULT_THREAD_FACTORY = new DefaultThreadFactory();

    private final long keepAliveTime;

    private final TimeUnit timeUnit;

    /**
     * 构造函数参数
     *
     * @param initSize  初始的线程数量
     * @param maxSize   最大的线程数量
     * @param coreSize  核心线程数量
     * @param queueSize 任务队列的最大数量
     */
    public BasicThreadPool(int initSize, int maxSize, int coreSize,
                           int queueSize) {
        this(initSize, maxSize, coreSize, DEFAULT_THREAD_FACTORY,
                queueSize, DEFAULT_DENY_POLICY, 10, TimeUnit.SECONDS);
    }

    /**
     * 构造线程池需要传入的参数，更多
     *
     * @param initSize      初始的线程数量
     * @param maxSize       最大的线程数量
     * @param coreSize      核心线程数量
     * @param threadFactory 创建线程工厂
     * @param queueSize     任务队列的最大数量（最大任务数）
     * @param denyPolicy    任务队列满后的拒绝策略
     * @param keepAliveTime
     * @param timeUnit
     */
    public BasicThreadPool(int initSize, int maxSize, int coreSize,
                           ThreadFactory threadFactory, int queueSize,
                           DenyPolicy denyPolicy, long keepAliveTime, TimeUnit timeUnit) {
        this.initSize = initSize;
        this.maxSize = maxSize;
        this.coreSize = coreSize;
        this.threadFactory = threadFactory;
        this.runnableQueue = new LinkedRunnableQueue(queueSize, denyPolicy, this);
        this.keepAliveTime = keepAliveTime;
        this.timeUnit = timeUnit;
        this.init();
    }

    /**
     * 初始化时，先创建initSize 个线程
     */
    private void init() {
        start();
        for (int i = 0; i < initSize; i++) {
            newThread();
        }
    }

    /**
     * 提交任务，只需要将 runnable插入任务队列
     *
     * @param runnable
     */
    @Override
    public void execute(Runnable runnable) {
        if (this.isShutdown)
            throw new IllegalStateException("The thread pool is destroy");
        // 提交任务
        this.runnableQueue.offer(runnable);
    }

    /**
     * 线程池的自动维护，具体逻辑看 InternalTask的run方法
     */
    private void newThread() {
        InternalTask internalTask = new InternalTask(runnableQueue);
        Thread thread = this.threadFactory.createThread(internalTask);
        ThreadTask threadTask = new ThreadTask(thread, internalTask);
        threadQueue.offer(threadTask);
        this.activeCount++;
        thread.start();
    }

    /**
     *
     */
    private void removeThread() {
        //从线程池中 remove 某个线程
        ThreadTask threadTask = threadQueue.remove();
        threadTask.internalTask.stop();
        this.activeCount--;
    }


    @Override
    public void run() {
        // 主要用于维护线程数量，比如 扩容、回收等工作
        while (!isShutdown && !isInterrupted()) {
            try {
                timeUnit.sleep(keepAliveTime);
            } catch (InterruptedException e) {
                isShutdown = true;
                break;
            }

            synchronized (this) {
                if (isShutdown)
                    break;
                System.out.println(runnableQueue.size() + "==" + activeCount);
                // 当前队列中 有任务尚未处理，并且 activeCount< coreSize 则继续扩容
                if (runnableQueue.size() > 0 && activeCount < coreSize) {
                    for (int i = initSize; i < coreSize; i++) {
                        System.out.println("--create");
                        newThread();
                    }
                    continue;
                }
                // 当前队列中 有任务尚未处理，并且 activeCount< maxSize 则继续扩容
                if (runnableQueue.size() > 0 && activeCount < maxSize) {
                    for (int i = coreSize; i < maxSize; i++) {
                        newThread();
                    }
                }
                // 如果任务队列中 没有任务，则需要回收，回收至 coreSize 即可
                if (runnableQueue.size() == 0 && activeCount > maxSize) {
                    System.out.println("remove...");
                    for (int i = coreSize; i < maxSize; i++) {
                        removeThread();
                    }
                }
            }
        }
    }

    @Override
    public void shutdown() {
        synchronized (this) {
            if (isShutdown) return;
            isShutdown = true;
            threadQueue.forEach(threadTask ->
            {
                threadTask.internalTask.stop();
                threadTask.thread.interrupt();
            });
            this.interrupt();
        }
    }

    @Override
    public int getInitSize() {
        if (isShutdown)
            throw new IllegalStateException("The thread pool is destroy");
        return this.initSize;
    }

    @Override
    public int getMaxSize() {
        if (isShutdown)
            throw new IllegalStateException("The thread pool is destroy");
        return this.maxSize;
    }

    @Override
    public int getCoreSize() {
        if (isShutdown)
            throw new IllegalStateException("The thread pool is destroy");
        return this.coreSize;
    }

    @Override
    public int getQueueSize() {
        if (isShutdown)
            throw new IllegalStateException("The thread pool is destroy");
        return runnableQueue.size();
    }

    @Override
    public int getActiveCount() {
        synchronized (this) {
            return this.activeCount;
        }
    }

    @Override
    public boolean isShutdown() {
        return this.isShutdown;
    }

    private static class DefaultThreadFactory implements ThreadFactory {

        private static final AtomicInteger GROUP_COUNTER = new AtomicInteger(1);

        private static final ThreadGroup group = new ThreadGroup("MyThreadPool-" + GROUP_COUNTER.getAndDecrement());

        private static final AtomicInteger COUNTER = new AtomicInteger(0);

        @Override
        public Thread createThread(Runnable runnable) {
            return new Thread(group, runnable, "thread-pool-" + COUNTER.getAndDecrement());
        }
    }

    /**
     * ThreadTask 只是 InternalTask和Thread的一个组合
     */
    private static class ThreadTask {
        public ThreadTask(Thread thread, InternalTask internalTask) {
            this.thread = thread;
            this.internalTask = internalTask;
        }

        Thread thread;

        InternalTask internalTask;
    }
}
```

- BasicThreadPool 同时也是Thread的子类，它在初始化的时候启动，在keepalive时间到了之后，再自动维护活动线程数量
- TODO： BasicThreadPool 采用继承Thread的方式，不好的方式，会暴露Thread的方法，建议 改为组合关系，TODO 后面我这边会自行修改 

### 线程自动维护

- 自动维护线程的代码块（run方法） 是同步代码块，主要是为了阻止在线程维护过程中 线程池销毁引起的数据不一致的问题
- 任务队列中若存在积压任务，并且当前活动线程少于核心线程数，则新建 （CoreSize-initSize）数量的线程，并且将其假如到活动线程队列中，为了防止马上进行 （maxSize-coreSize）数量的扩充，建议使用 continue 终止本次循环
- 任务队列中有 积压任务，并且当前活动线程少于 最大线程数，则新建（maxSIze-coreSIze）数量的扩充，建议使用 continue 终止本次循环
- 当线程池不够繁忙时，则需要回收部分线程，回收到coreSize 数量即可，回收时调用removeThread（）方法，在该方法中需要考虑的一点是，如果被回收的线程恰巧从Runnable任务取出了某个任务，则会继续保持该线程的运行，知道完成了任务的运行为止 。详情见 InterTask 的run方法

### 线程池销毁shutdown

- 线程池的销毁同样需要同步机制的保护，主要是 为了防止与线程池本身的维护线程引起数据冲突



# 线程池的应用

```java
public class ThreadPoolTest {
    public static void main(String[] args) throws InterruptedException {
        final ThreadPool threadPool = new BasicThreadPool(2, 6, 4, 1000);
        for (int i = 0; i < 20; i++)
            threadPool.execute(() ->
            {
                try {
                    TimeUnit.SECONDS.sleep(10);
                    System.out.println(Thread.currentThread().getName() + " is running and done.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });

//        for (; ; )
//        {
//            System.out.println("getActiveCount:" + threadPool.getActiveCount());
//            System.out.println("getQueueSize:" + threadPool.getQueueSize());
//            System.out.println("getCoreSize:" + threadPool.getCoreSize());
//            System.out.println("getMaxSize:" + threadPool.getMaxSize());
//            System.out.println("======================================");
//            TimeUnit.SECONDS.sleep(5);
//        }

        TimeUnit.SECONDS.sleep(12);
        threadPool.shutdown();

        Thread.currentThread().join();
    }
}
```

# 总结：

TODO： 看看 JDK 的 ExecutorService的原理和源码，据说是类似的

