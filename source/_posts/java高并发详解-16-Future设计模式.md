---
title: java高并发详解-16-Future设计模式
copyright: true
comments: true
related_posts: true
date: 2021-04-07 22:37:12
tags: Future设计模式
categories: 高并发详解
---

# Future设计模式



下面是Future设计模式涉及的关键接口以及它们之间的关系UML 图

![](/uploads/java-concurrency-master/design_future.png)

#  接口定义

## 1.Future接口定义

Future提供了获取计算结果和判断任务是否完成的两个接口，其中获取计算结果将会导致调用阻塞（在任务还未完成的情况下）

```java
public interface Future<T> {
    // 返回计算后的结果，该方法会陷入阻塞的状态
    T get() throws InterruptedException;

    // 判断任务是否 已经被执行完成
    boolean done();
}
```

## 2.FutureService接口设计

FutureService 主要用于提交任务，提交任务主要有2种，第一种不需要返回值，第二种则需要获得最终的计算结果。

```java
public interface FutureService<IN, OUT> {
    /**
     * 提交不需要返回值的任务 ，future.get方法返回的将会是 null
     *
     * @param runnable
     * @return
     */
    Future<?> submit(Runnable runnable);

    /**
     * 提交需要返回值的任务，其中task接口代替了RUnnable 接口
     *
     * @param task
     * @param input
     * @param callback
     * @return
     */
    Future<OUT> submit(Task<IN, OUT> task, IN input, Callback<OUT> callback);

    /**
     * 使用静态方法 创建一个FutureService 的实现
     *
     * @param <T>
     * @param <R>
     * @return
     */
    static <T, R> FutureService<T, R> newService() {
        return new FutureServiceImpl<>();
    }

}
```

## 3.Task接口设计

Task接口主要是提供给调用者实现计算逻辑使用的，可以接受一个参数并且返回最终的计算结果，这一点非常类似于 JDK1.5中的Callable接口，

```java
public interface Task<IN, OUT>
{
    /**
     * 给定一个参数，经过计算返回结果
     * @param input
     * @return
     */
    OUT get(IN input);
}
```

# 程序实现 

## FutureTask

FutureTask 是Future的一个实现，除了实现Future 中定义的方法，还额外增加了 finish ，该方法主要用于接收任务被完成的通知，

```java
public class FutureTask<T> implements Future<T> {   //计算结果
    private T result;
    //任务是否完成
    private boolean isDone = false;
    // 定义对象锁
    private final Object LOCK = new Object();


    @Override
    public T get() throws InterruptedException {
        synchronized (LOCK) {
            // 当任务还没有完成时，调用get方法会被挂起而进入阻塞
            while (!isDone) {
                LOCK.wait();
            }
            // 返回最终计算结果
            return result;
        }
    }

    /**
     * finish 方法主要用于为 FutureTask 设置计算结果
     *
     * @param result
     */
    protected void finish(T result) {
        synchronized (LOCK) {
            // balking设计模式
            if (isDone)
                return;
            //计算完成，为result 指定结果，并且将isDone 设为true，同时唤醒阻塞中的线程
            this.result = result;
            this.isDone = true;
            LOCK.notifyAll();
        }
    }

    /**
     * 返回当前任务是否已经完成
     *
     * @return
     */
    @Override
    public boolean done() {
        return isDone;
    }
}
```



## FutureServiceImpl

```java
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 主要作用在于当提交任务时，创建一个新的线程来受理该任务，进而达到任务异步执行的效果
 *
 * @param <IN>
 * @param <OUT>
 */
public class FutureServiceImpl<IN, OUT> implements FutureService<IN, OUT> {
    // 为执行的线程执行名字前缀，（再三强调，为线程起一个特殊的名字是一个非常好的编程习惯）
    private final static String FUTURE_THREAD_PREFIX = "FUTURE-";

    private final AtomicInteger nextCounter = new AtomicInteger(0);

    private String getNextName() {
        return FUTURE_THREAD_PREFIX + nextCounter.getAndIncrement();
    }

    @Override
    public Future<?> submit(Runnable runnable) {
        final FutureTask<Void> future = new FutureTask<>();
        new Thread(() ->
        {
            runnable.run();
            // 任务执行结束之后将null作为结果传给future
            future.finish(null);
        }, getNextName()).start();

        return future;
    }

    @Override
    public Future<OUT> submit(Task<IN, OUT> task, IN input, Callback<OUT> callback) {
        final FutureTask<OUT> future = new FutureTask<>();
        new Thread(() ->
        {
            OUT result = task.get(input);
            // 任务执行结束之后，将真实的结果通过finish方法 传递给future
            future.finish(result);
            if (null != callback)
                callback.call(result);
        }, getNextName()).start();

        return future;
    }
}
```

## 测试类：

```java
public class FutureTest {
    public static void main(String[] args)
            throws InterruptedException {
        FutureService<String, Integer> service = FutureService.newService();
        service.submit(input ->
        {
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return input.length();
        }, "Hello", System.out::println);
    }
}
```

# 总结

todo 优化

- 将提交的任务交给线程池运行，
- get方法没有超时功能，如果获取一个计算结果在规定时间内没有返回，则可以抛出异常通知调用线程
- Future未提供Cancel功能，当任务提交之后还可以对其进行取消
- 任务运行时出错未提供回调方式

