---
title: 深入理解jvm-03-垃圾收集器
copyright: true
related_posts: true
date: 2020-12-26 23:59:09
tags: 垃圾收集器
categories: jvm
---

# 垃圾收集器

## 垃圾收集需要完成的3件事情

- 哪些内存需要回收
- 什么时候回收
- 如果回收

## 为什么要了解gc 和内存分配：

当需要排查各种内存溢出、内存泄漏问题时，当垃圾收集成为系统达到更高并发量的瓶颈时，我们就必须对这些“自动化”的技术实施必要的监控和调节。

## 运行时数据区域：

### 线程隔离区域

- 程序计数器、虚拟机栈、本地方法栈3个区域随线程而生，随线程而灭，栈中的栈帧随着方法的进入和退出而有条不紊地执行着出栈和入栈操作。

- 每一个栈帧中分配多少内存基本上是在类结构确定下来时就已知的（尽管在运行期会由即时编译器进行一些优化，但在基于概念模型的讨论里，大体上可以认为是编译期可知的），因此这几个区域的内存分配和回收都具备确定性，在这几个区域内就不需要过多考虑如何回收的问题，
- 当方法结束或者线程结束时，内存自然就跟随着回收了。

### 线程共享区域

- Java堆和方法区这两个区域则有着很显著的不确定性：一个接口的多个实现类需要的内存可能会不一样，一个方法所执行的不同条件分支所需要的内存也可能不一样，

- 只有处于运行期间，我们才能知道程序究竟会创建哪些对象，创建多少个对象，这部分内存的分配和回收是动态的

- 所以 GC 关注的就是这部分区域的内存管理 （分配和回收）

  

## 判断对象“死亡”

在堆里面存放着Java世界中几乎所有的对象实例，垃圾收集器在对堆进行回收前，第一件事情就是要确定这些对象之中哪些还“存活”着，哪些已经“死去”（“死去”即不可能再被任何途径使用的对象）了。

### 引用计数法

- 在对象中添加一个***引用计数器***，
- 每当有一个地方引用它时，计数器值就加一；
- 当引用失效时，计数器值就减一；任何时刻计数器为零的对象就是不可能再被使用的



主流的Java虚拟机里面都没有选用引用计数算法来管理内存，主要原因是，这个看似简单的算法有很多例外情况要考虑，必须要配合大量额外处理才能保证正确地工作，譬如单纯的引用计数就很难解决对象之间相互循环引用的问题。



举个简单的例子，请看代码中的testGC()方法：对象objA和objB都有字段instance，赋值令objA.instance=objB及objB.instance=objA，除此之外，这两个对象再无任何引用，实际上这两个对象已经不可能再被访问，但是它们因为互相引用着对方，导致它们的引用计数都不为零，引用计数算法也就无法回收它们。

```java
/**
 * testGC()方法执行后，objA和objB会不会被GC呢？
 *
 * @author zzm
 */
public class ReferenceCountingGC {

    public Object instance = null;

    private static final int _1MB = 1024 * 1024;

    /**
     * 这个成员属性的唯一意义就是占点内存，以便在能在GC日志中看清楚是否有回收过
     */
    private byte[] bigSize = new byte[2 * _1MB];

    public static void testGC() {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;

        objA = null;
        objB = null;

        // 假设在这行发生GC，objA和objB是否能被回收？
        System.gc();
    }

    public static void main(String[] args) {
        testGC();
    }
}

结果： 
[GC (System.gc()) [PSYoungGen: 14622K->1008K(153088K)] 14622K->1016K(502784K), 0.0007592 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 153088K, used 1008K [0x0000000715980000, 0x0000000720400000, 0x00000007c0000000)
  eden space 131584K, 0% used [0x0000000715980000,0x0000000715980000,0x000000071da00000)
  from space 21504K, 4% used [0x000000071da00000,0x000000071dafc040,0x000000071ef00000)
  to   space 21504K, 0% used [0x000000071ef00000,0x000000071ef00000,0x0000000720400000)
 ParOldGen       total 349696K, used 8K [0x00000005c0c00000, 0x00000005d6180000, 0x0000000715980000)
  object space 349696K, 0% used [0x00000005c0c00000,0x00000005c0c02000,0x00000005d6180000)
 Metaspace       used 3118K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 338K, capacity 388K, committed 512K, reserved 1048576K
}

```

从运行结果中可以清楚看到内存回收日志，意味着虚拟机并没有因为这两个对象互相引用就放弃回收它们，这也从侧面说明了Java虚拟机并不是通过引用计数算法来判断对象是否存活的。



### 可达性分析算法

- 基本思路就是通过一系列称为“GC Roots”的根对象作为起始节点集，从这些节点开始，根据引用关系向下搜索，搜索过程所走过的路径称为**“引用链”（Reference Chain）**

- **如果某个对象到GC Roots间没有任何引用链相连**，或者用图论的话来说就是从GC Roots到这个对象不可达时，则证明此对象是不可能再被使用的。

![](/uploads/jvm/04-gcRoot.png)

在Java技术体系里面，固定可作为***GC Roots***的对象包括以下几种：

- 在虚拟机栈（栈帧中的本地变量表）中引用的对象，譬如各个线程被调用的方法堆栈中使用到的参数、局部变量、临时变量等。在方法区中类静态属性引用的对象，譬如Java类的引用类型静态变量。
- 在方法区中常量引用的对象，譬如字符串常量池（String Table）里的引用。
- 在本地方法栈中JNI（即通常所说的Native方法）引用的对象。
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器
- 所有被同步锁（synchronized关键字）持有的对象。
- 反映Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等。

ps:  当只针对 java堆中的某一块区域发起GC的时候，  使用局部回收 或者 分代回收 ，需要想到 各个区域并不是孤立封闭的，某个区域里 的对象很有可能被 堆中的其他区域对象引用，所以 这些关联区域的对象也要一起 加入到 GC Roots集合中 去，才能保证 可达性分析的正确性



## 引用 reference

- 无论是通过计数器法 判断对象的引用数量，还是通过可达性分析算法 判断对象是否引用链可达。 判断对象是否存活 都和 “引用” 离不开关系 。

- 如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称该reference数据是代表某块内存、某个对象的引用

### 强引用（Strongly Re-ference）

- 最传统的“引用”的定义，是指在程序代码之中普遍存在的引用赋值，即类似“Objectobj=new Object()”这种引用关系。

- 无论任何情况下，只要强引用关系还存在，垃圾收集器就永远不会回收掉被引用的对象。

### 软引用（Soft Reference）

- 用来描述一些还有用，但非必须的对象。只被软引用关联着的对象，

- 在系统将要发生内存溢出异常前，会把这些对象列进回收范围之中进行第二次回收，

- 如果这次回收还没有足够的内存，才会抛出内存溢出异常

### 弱引用（Weak Reference）

- 用来描述那些非必须对象，但是它的强度比软引用更弱一些，
- 被弱引用关联的对象只能生存到下一次垃圾收集发生为止。
- 当垃圾收集器开始工作，无论当前内存是否足够，都会回收掉只被弱引用关联的对象

### 虚引用（Phantom Reference）



## 生存还是死亡？

即使在可达性分析算法中判定为不可达的对象，也不是“非死不可”的，这时候它们暂时还处于“缓刑”阶段，

要真正宣告一个对象死亡，至少要经历两次标记过程：

- 如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记，

- 随后进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。

- 假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过，那么虚拟机将这两种情况都视为“没有必要执行”。



如果这个对象被判定为确有必要执行finalize()方法，那么该对象将会被放置在一个名为F-Queue的队列之中，并在稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行它们的finalize()方法。这里所说的“执行”是指虚拟机会触发这个方法开始运行，但并不承诺一定会等待它运行结束。



原因是，如果某个对象的finalize()方法执行缓慢，或者更极端地发生了死循环，将很可能导致F-Queue队列中的其他对象永久处于等待，甚至导致整个内存回收子系统的崩溃。finalize()方法是对象逃脱死亡命运的最后一次机会，稍后收集器将对F-Queue中的对象进行第二次小规模的标记，如果对象要在finalize()中成功拯救自己——只要重新与引用链上的任何一个对象建立关联即可，譬如把自己（this关键字）赋值给某个类变量或者对象的成员变量，那在第二次标记时它将被移出“即将回收”的集合；如果对象这时候还没有逃脱，那基本上它就真的要被回收了



```java
/**
 * 此代码演示了两点：
 * 1.对象可以在被GC时自我拯救。
 * 2.这种自救的机会只有一次，因为一个对象的finalize()方法最多只会被系统自动调用一次
 *
 * @author zzm
 */
public class FinalizeEscapeGC {

    public static FinalizeEscapeGC SAVE_HOOK = null;

    public void isAlive() {
        System.out.println("yes, i am still alive :)");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize method executed!");
        FinalizeEscapeGC.SAVE_HOOK = this;
    }

    public static void main(String[] args) throws Throwable {
        SAVE_HOOK = new FinalizeEscapeGC();

        //对象第一次成功拯救自己
        SAVE_HOOK = null;
        System.gc();
        // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }

        // 下面这段代码与上面的完全相同，但是这次自救却失败了
        SAVE_HOOK = null;
        System.gc();
        // 因为Finalizer方法优先级很低，暂停0.5秒，以等待它
        Thread.sleep(500);
        if (SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no, i am dead :(");
        }
    }
}


执行结果：
    finalize method executed!
yes, i am still alive :)
no, i am dead :(
```

运行结果可以看到，

- SAVE_HOOK对象的finalize()方法确实被垃圾收集器触发过，并且在被收集前成功逃脱了。
- 另外一个值得注意的地方就是，代码中有两段完全一样的代码片段，执行结果却是一次逃脱成功，一次失败了。
- 这是因为任何一个对象的finalize()方法都只会被系统自动调用一次，如果对象面临下一次回收，它的finalize()方法不会被再次执行，因此第二段代码的自救行动失败了。



## 回收方法区

- 方法区垃圾收集的“性价比”通常也是比较低的：

在Java堆中，尤其是在新生代中，对常规应用进行一次垃圾收集通常可以回收70%至99%的内存空间，相比之下，方法区回收囿于苛刻的判定条件，其区域垃圾收集的回收成果往往远低于此。

- 方法区的垃圾收集主要回收两部分内容：**废弃的常量** 和**不再使用的类型**



关于是否要对类型进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制，还可以使用-verbose：class以及-XX：+TraceClass-Loading、

-XX：+TraceClassUnLoading查看类加载和卸载信息，其中-verbose：class和-XX：+TraceClassLoading可以在Product版的虚拟机中使用，



- 在大量使用反射、动态代理、CGLib等字节码框架，动态生成JSP以及OSGi这类频繁自定义类加载器的场景中，通常都需要Java虚拟机具备类型卸载的能力，以保证不会对方法区造成过大的内存压
  力。