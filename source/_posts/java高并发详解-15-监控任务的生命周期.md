---
title: java高并发详解-15-监控任务的生命周期
copyright: true
comments: true
related_posts: true
date: 2021-03-09 16:20:40
tags: 监控任务的生命周期
categories: 高并发详解
---

- 这个例子我已经用到了实际工作中，怎么说呢，效果还不错吧
- 既监控了任务的状态，也实现了单一职责，将任务task的逻辑 与监控 逻辑进行了分离
- 也方便统一 任务的状态，可以落数据库



## 1.Observable接口定义

```java
public interface Observable {
    // 任务生命周期的枚举类型
    enum Cycle {
        STARTED, RUNNING, DONE, ERROR
    }

    // 获取当前任务的生命周期状态
    Cycle getCycle();

    // 定义启动线程的方法，作用是为了屏蔽 Thread的其他方法
    void start();

    //定义线程的打断方法，作用于 start方法一样，也是为了屏蔽Thread的其他方法
    void interrupt();
}

```



## 2.TaskLifecycle 事件回调者

```java
public interface TaskLifecycle<T>
{
    void onStart(Thread thread);

    void onRunning(Thread thread);

    void onFinish(Thread thread, T result);

    void onError(Thread thread, Exception e);
    
}
```

## 3.Task函数接口定义

```java

@FunctionalInterface
public interface Task<T> {
    // 任务执行接口，该接口允许有返回值 
    T call();
}
```



## 4.ObservableThread 事件发起者

```java
public class ObservableThread<T> extends Thread implements Observable {

    private final TaskLifecycle<T> lifecycle;
    private final Task<T> task;
    private Cycle cycle;
    public ObservableThread(TaskLifecycle<T> lifecycle, Task<T> task) {
        super();
        if (task == null)
            throw new IllegalArgumentException("The task is required.");
        this.lifecycle = lifecycle;
        this.task = task;
    }

    @Override
    public final void run() {
        this.update(Cycle.STARTED, null, null);
        try {
            this.update(Cycle.RUNNING, null, null);
            T result = this.task.call();
            this.update(Cycle.DONE, result, null);
        } catch (Exception e) {
            this.update(Cycle.ERROR, null, e);
        }
    }

    private void update(Cycle cycle, T result, Exception e) {
        if (lifecycle == null)
            return;
        this.cycle = cycle;
        try {
            switch (cycle) {
                case STARTED:
                    this.lifecycle.onStart(currentThread());
                    break;
                case RUNNING:
                    this.lifecycle.onRunning(currentThread());
                    break;
                case DONE:
                    this.lifecycle.onFinish(currentThread(), result);
                    break;
                case ERROR:
                    this.lifecycle.onError(currentThread(), e);
                    break;
            }
        } catch (Exception ex) {
            if (cycle == Cycle.ERROR) {
                throw ex;
            }
        }
    }

    @Override
    public Cycle getCycle() {
        return this.cycle;
    }
}
```



