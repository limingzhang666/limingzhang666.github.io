---
title: java高并发详解-06-ThreadGroup详解
copyright: true
comments: true
related_posts: true
date: 2021-02-23 00:08:17
tags: ThreadGroup
categories: 高并发详解
---

- 创建线程的时候如果没有显示得指定ThreadGroup，那么新的线程会被加入到父线程相同的ThreadGroup中
- 其实这里主要就是看ThreadGroup的源码，以及Thread的源码，就可以知道关于Thread的一些规则，比如 Thread的优先级 的范围，以及不能超过ThreadGroup的MAX



# ThreadGroup 与Thread

## 创建ThreadGroup

```java
// 1.    
public ThreadGroup(String name) {
        this(Thread.currentThread().getThreadGroup(), name);
    }
//2.
public ThreadGroup(ThreadGroup parent, String name) {
     this(checkParentAccess(parent), parent, name);
}

//这个是private ，不对外暴露
    private ThreadGroup(Void unused, ThreadGroup parent, String name) {
        this.name = name;
        this.maxPriority = parent.maxPriority;
        this.daemon = parent.daemon;
        this.vmAllowSuspension = parent.vmAllowSuspension;
        this.parent = parent;
        parent.add(this);
    }
```

- 第一个构造函数为 ThreadGroup赋予了名字，它的默认父 ThreadGroup是创建它的线程所在的ThreadGroup
- 第二构造函数赋予group名字的同时，显示指定了父ThreadGroup

## 复制Thread数组和ThreadGroup数组

### 复制Thread数组

```java
 /**
     * Copies into the specified array every active thread in this
     * thread group and its subgroups.
     *
     * <p> An invocation of this method behaves in exactly the same
     * way as the invocation
     *
     * <blockquote>
     * {@linkplain #enumerate(Thread[], boolean) enumerate}{@code (list, true)}
     * </blockquote>
     *
     * @param  list
     *         an array into which to put the list of threads
     *
     * @return  the number of threads put into the array
     *
     * @throws  SecurityException
     *          if {@linkplain #checkAccess checkAccess} determines that
     *          the current thread cannot access this thread group
     *
     * @since   JDK1.0
     */
    public int enumerate(Thread list[]) {
        checkAccess();
        return enumerate(list, 0, true);
    }

 /**
     * Copies into the specified array every active thread in this
     * thread group. If {@code recurse} is {@code true},
     * this method recursively enumerates all subgroups of this
     * thread group and references to every active thread in these
     * subgroups are also included. If the array is too short to
     * hold all the threads, the extra threads are silently ignored.
     *
     * <p> An application might use the {@linkplain #activeCount activeCount}
     * method to get an estimate of how big the array should be, however
     * <i>if the array is too short to hold all the threads, the extra threads
     * are silently ignored.</i>  If it is critical to obtain every active
     * thread in this thread group, the caller should verify that the returned
     * int value is strictly less than the length of {@code list}.
     *
     * <p> Due to the inherent race condition in this method, it is recommended
     * that the method only be used for debugging and monitoring purposes.
     *
     * @since   JDK1.0
     */
    public int enumerate(Thread list[], boolean recurse) {
        checkAccess();
        return enumerate(list, 0, recurse);
    }

// 底层私有逻辑
    private int enumerate(Thread list[], int n, boolean recurse) {
        int ngroupsSnapshot = 0;
        ThreadGroup[] groupsSnapshot = null;
        synchronized (this) {
            if (destroyed) {
                return 0;
            }
            int nt = nthreads;
            if (nt > list.length - n) {
                nt = list.length - n;
            }
            for (int i = 0; i < nt; i++) {
                if (threads[i].isAlive()) {
                    list[n++] = threads[i];
                }
            }
            if (recurse) {
                ngroupsSnapshot = ngroups;
                if (groups != null) {
                    groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
                } else {
                    groupsSnapshot = null;
                }
            }
        }
        if (recurse) {
            for (int i = 0 ; i < ngroupsSnapshot ; i++) {
                n = groupsSnapshot[i].enumerate(list, n, true);
            }
        }
        return n;
    }
```

- 上面的public 的2个方法，会将ThreadGroup 中active线程全部复制到Thread数组中，
- 其中recurse参数如果为True，则方法会将所有子 group的active线程全部递归到Thread数组中，
- enumerate（list ）实际上等价于 enumerate（list，true）



### 复制ThreadGroup数组

```java
 public int enumerate(ThreadGroup list[]) {
        checkAccess();
        return enumerate(list, 0, true);
    }

     
    public int enumerate(ThreadGroup list[], boolean recurse) {
        checkAccess();
        return enumerate(list, 0, recurse);
    }
```

- 和复制Thread数组类似，上面方法用于复制当前ThreadGroup的子 Group，同样recurse会决定是否以递归的方式复制



## ThreadGroup操作

Threadgroup 并不能提供对线程的管理，主要功能是对线程提供组织

### ThreadGroup基本操作

- activeCount（）： 用于获取group中活跃的线程，只是一个估计值，该方法会递归获取其他子group中的活跃线程
- activeGroupCount（）： 用于获取group中活跃的子group，只是一个估计值，该方法也会递归获取所有的子 group
- getMaxPriority() 用于获取group的优先级，默认情况下，Group的优先级为10，在该group中，所有线程的优先级不能大于group的优先级
- getName() 用于获取group的名字
- getParent（）用于获取group的父 group，如果父group不存在，则会返回null ，比如system group的父group就是 null
- list（）该方法没有返回值，执行该方法会将group中所有的活跃线程信息全部输出到控制台，也就是 System.out
- parentOf（ThreadGroup g）会判断当前group是不是给定的group的父group ，另外如果给定的group就是本身，那么也返回true
- setMaxPriority(int pri)会指定group的最大优先级 ，最大优先级不能超过父 group的最大优先级。执行该方法不仅仅会改变当前group的最大优先级，还会改变所有子goup的最大优先级



ThreadGroup的interrupt

```java
  /**
     * Interrupts all threads in this thread group.
     * <p>
     * First, the <code>checkAccess</code> method of this thread group is
     * called with no arguments; this may result in a security exception.
     * <p>
     * This method then calls the <code>interrupt</code> method on all the
     * threads in this thread group and in all of its subgroups.
     *
     * @exception  SecurityException  if the current thread is not allowed
     *               to access this thread group or any of the threads in
     *               the thread group.
     * @see        java.lang.Thread#interrupt()
     * @see        java.lang.SecurityException
     * @see        java.lang.ThreadGroup#checkAccess()
     * @since      1.2
     */
    public final void interrupt() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            checkAccess();
            for (int i = 0 ; i < nthreads ; i++) {
                threads[i].interrupt();
            }
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            groupsSnapshot[i].interrupt();
        }
    }

```

- interrupt一个thread group会导致 该group中所有的active 线程都被interrupt ，
- 也就说该 group中的每一个interrupt 标识都被设置了 
- 通过源码分析，可以看出interrupt内部会执行所有thread 的interrupt方法，并且会递归获取子 group，然后执行他们各自的interrupt方法

### ThreadGroup的destory

```java

    /**
     * Destroys this thread group and all of its subgroups. This thread
     * group must be empty, indicating that all threads that had been in
     * this thread group have since stopped.
     * <p>
     * First, the <code>checkAccess</code> method of this thread group is
     * called with no arguments; this may result in a security exception.
     *
     * @exception  IllegalThreadStateException  if the thread group is not
     *               empty or if the thread group has already been destroyed.
     * @exception  SecurityException  if the current thread cannot modify this
     *               thread group.
     * @see        java.lang.ThreadGroup#checkAccess()
     * @since      JDK1.0
     */
    public final void destroy() {
        int ngroupsSnapshot;
        ThreadGroup[] groupsSnapshot;
        synchronized (this) {
            checkAccess();
            if (destroyed || (nthreads > 0)) {
                throw new IllegalThreadStateException();
            }
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
            if (parent != null) {
                destroyed = true;
                ngroups = 0;
                groups = null;
                nthreads = 0;
                threads = null;
            }
        }
        for (int i = 0 ; i < ngroupsSnapshot ; i += 1) {
            groupsSnapshot[i].destroy();
        }
        if (parent != null) {
            parent.remove(this);
        }
    }
```

- destory 用于销毁 ThreadGroup ，该方法只是针对一个没有任何active线程的group 进行一个 destory标记，调用该方法的结果是在父 group中将自己移除

### 守护ThreadGroup

- 线程可以设置为守护线程，ThreadGroup也可以设置为 守护ThreadGroup，
- 但是将一个THreadGroup 设置为 daemon ，也并不会影响线程的daemon属性
- 如果一个 ThreadGroup 的daemon被设置为  true，那么在group中没有任何active线程的时候，该group将自动destory 