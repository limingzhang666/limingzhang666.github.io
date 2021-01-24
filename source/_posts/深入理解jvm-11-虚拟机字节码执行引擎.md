---
title: 深入理解jvm-11-虚拟机字节码执行引擎
copyright: true
related_posts: true
date: 2021-01-19 23:42:20
tags: 虚拟机字节码执行引擎
categories: jvm
---

# 虚拟机字节码执行引擎

代码编译的结果从本地机器码转变为字节码，是存储格式发展的一小步，却是编程语言发展的一大步。



## 概述

- 执行引擎是Java虚拟机核心的组成部分之一。

- “虚拟机”是一个相对于“物理机”的概念，这两种机器都有代码执行能力，

- 其区别是物理机的执行引擎是直接建立在处理器、缓存、指令集和操作系统层面上的，

- 而虚拟机的执行引擎则是由软件自行实现的，因此可以不受物理条件制约地定制指令集与执行引擎的结构体系，能够执行那些不被硬件直接支持的指令集格式。



## 运行时栈帧结构

### 栈帧（Stack Frame）

- Java虚拟机以方法作为最基本的执行单元，而 **栈帧** （Stack Frame）则是用于java虚拟机进行方法调用和方法执行背后的数据结构， 它也是虚拟机运行时数据区中的虚拟机栈（Virtual Machine Stack）的栈元素。

- 栈帧存储了方法的**局部变量表**、**操作数栈**、**动态连接**和**方法返回地址** 等信息，
- 每一个方法从调用开始至执行结束的过程，都对应着一个栈帧在虚拟机栈里面从**入栈到出栈**的过程
- 在编译Java程序源码的时候，栈帧需要多大的局部变量表，需要多深的操作数栈就已经被分析计算出来，并且写入到方法表的Code属性之中。
- 一个栈帧需要分配多少内存，并不会收到程序运行期变量数据的影响，而仅仅取决于程序源码和具体的虚拟机实现的栈内存布局形式



- 一个线程中的方法调用链 可能会很长，以java程序的角度来看，同一时刻、同一条线程里面，在调用堆栈的所有方法都同时处于执行状态。
- 而对于执行引擎来讲，在活动线程中，只有位于栈顶的方法才是运行的，只有位于栈顶的栈帧才是生效的，其被称为“当前栈帧”（Current Stack Frame）,与这个栈帧关联的方法 被称为 “当前方法”（Current Method）。
- 执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作，

### 栈帧的概念结构

![](/uploads/jvm/10StackFrame/01-StackFrameConstruct.png)

####  局部变量表（Local Variables Table）

- 局部变量表 是一组变量值的存储空间，用于存放方法参数和方法内部定义的局部变量表。 

- 在Java程序被编译成Class文件时，就在方法的Code属性中的**max_locals** 数据项中确定了该方法所需分配的局部变量表的最大容量

- 局部变量表的容量以 变量槽（Variable slot）为最小单位，

- 《Java虚拟机规范》被没有明确指出一个变量槽应占用的 内存空间大小，只有向导性的说每个变量槽 Variable Slot 都应该能存放一个  boolean 、 byte、 char、 short、 int、 float、 reference 或看return Address类型的数据

##### 局部变量表中的数据类型

- 一个变量槽可以存放一个32位以内的数据类型，Java中占用不超过32位存储空间的数据类型有boolean、byte、char、short、int、float、reference[1]和returnAddress这8种类型。
- 而第7种reference类型表示对一个对象实例的引用，《Java虚拟机规范》既没有说明它的长度，也没有明确指出这种引用应有怎样的结构。但是一般来说，虚拟机实现至少都应当能通过这个引用做到两件事情，一是从根据引用直接或间接地查找到对象在Java堆中的数据存放的起始地址或索引，二是根据引用直接或间接地查找到对象所属数据类型在方法区中的存储的类型信息，否则将无法实现《Java语言规范》中定义的语法约定
- 第8种returnAddress类型目前已经很少见了，它是为字节码指令jsr、jsr_w和ret服务的，指向了一条字节码指令的地址，某些很古老的Java虚拟机曾经使用这几条指令来实现异常处理时的跳转，但现在也已经全部改为采用异常表来代替了
- 对于64位的数据类型，Java虚拟机会以 **高位对齐** 的方式为其分配 **两个连续的变量槽空间** 。Java语言中明确的64位的数据类型只有 long 和 double 两种。
- 这里把long和double数据类型分割存储的做法与“long和double的非原子性协定”中允许把一次long和double数据类型读写分割为两次32位读写的做法有些类
  似。
- 由于局部变量表是建立在线程堆栈中的，属于线程私有的数据，无论读写两个连续的变量槽是否为原子操作，都不会引起数据竞争和线程安全问题。



- Java虚拟机通过**索引定位**的方式使用局部变量表，索引值的范围是从0开始至局部变量表最大的变量槽数量。如果访问的是32位数据类型的变量，索引N就代表了使用第N个变量槽，如果访问的是64位数据类型的变量，则说明会**同时使用第N和N+1两个变量槽**.
- 对于两个相邻的共同存放一个64位数据的两个变量槽，虚拟机**不允许采用任何方式单独访问其中的某一个**，《Java虚拟机规范》中明确要求了如果遇到进行这种操作的字节码序列，虚拟机就应该在类加载的校验阶段中抛出异常。

- 当一个方法被调用时，Java虚拟机会使用局部变量表来完成**参数值到参数列表**的传递过程，即 **实参到形参**的传递。如果执行的时实例方法（没有被 static 修饰的方法），那局部变量表中的**第0 位索引**的变量槽 默认是 用于传递**方法实例的引用**，在方法中可以通过 关键字“this” 来访问到这个隐含的参数。 其余的参数 则是按照参数表顺序排列，占用从1开始的局部变量槽，参数表分配完毕后，在根据方法体内部定义的变量顺序和作用域分配其余的变量槽。

##### 局部变量表的重用

- 为了尽可能的节省栈帧耗用的内存空间，**局部变量表中的变量槽是可以重用**的，方法体中定义的变量，其作用域并不一定会覆盖整个方法体（就是一个方法体中的局部变量可能用到一半就不使用了，比如方法体有10行，而局部变量只在其中的3-4行被使用，那这个局部变量就没有覆盖整个方法体），如果当前字节码PC计数器的值 已经超过了某个变量的作用域，那这个变量对应的变量槽就可以交给其他变量来重用。
- 不过这样的设计除了节省栈帧空间外，还会伴随少量的**副作用**
- 例如 在某些情况下变量槽的复用会直接影响到系统的GC 行为，如下如代码所示

```java
/**
 * VM参数：-verbose:gc -Xms80M -Xmx80M -Xmn10M 
 * // -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
public class ReuseVariableSlot {
    public static void main(String[] args) {
        byte[] placeholder = new byte[64 * 1024 * 1024];
        System.gc();
    }
}


result: (我这里使用的JDK 9)
[0.028s][info][gc] Using G1
[0.142s][info][gc] GC(0) Pause Initial Mark (G1 Humongous Allocation) 2M->1M(80M) 1.368ms
[0.142s][info][gc] GC(1) Concurrent Cycle
[0.171s][info][gc] GC(1) Pause Remark 66M->66M(80M) 0.724ms
[0.171s][info][gc] GC(1) Pause Cleanup 66M->66M(80M) 0.174ms
[0.175s][info][gc] GC(2) Pause Full (System.gc()) 66M->66M(80M) 3.810ms
[0.175s][info][gc] GC(1) Concurrent Cycle 33.128ms

  
```

代码中可以看到：

向内存填充了64MB的数据，然后通知虚拟机进行垃圾收集。我们在虚拟机运行参数中加上“-verbose：gc”来看看垃圾收集的过程，发现在System.gc()运行后并没有回收掉这64MB的内存。

代码没有回收掉 placeHolder所占的内存说的过去，因为执行system.gc()时，变量 placeholder 还处于作用域之内，虚拟机自然不敢回收掉placeholder 的内存。

我们修改一下代码，改成这样

```java
/**
 * VM参数：-verbose:gc -Xms80M -Xmx80M -Xmn10M
 * // -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
public class ReuseVariableSlot {
    public static void main(String[] args) {
        {
            byte[] placeholder = new byte[64 * 1024 * 1024];
        }
        System.gc();
    }
}

result： 同上
    [0.026s][info][gc] Using G1
[0.137s][info][gc] GC(0) Pause Initial Mark (G1 Humongous Allocation) 2M->1M(80M) 1.593ms
[0.137s][info][gc] GC(1) Concurrent Cycle
[0.162s][info][gc] GC(1) Pause Remark 66M->66M(80M) 0.727ms
[0.165s][info][gc] GC(2) Pause Full (System.gc()) 66M->66M(80M) 3.594ms
[0.166s][info][gc] GC(1) Pause Cleanup 66M->66M(80M) 0.001ms
[0.166s][info][gc] GC(1) Concurrent Cycle 28.351ms
```

加入了花括号之后，placeholder的作用域被限制在了 花括号之内，从代码逻辑上讲，在执行 system.gc()的时候，placeholder 已经不可能再被访问了，但执行这段程序的时候，发现 64M的内存 还是没有被回收掉，

为什么呢，解释之前我们再修改一下代码,先对这段代码进行第二次修改，在调用System.gc()之前加入一行“int  a=0；”，

```bash
/**
 * VM参数：-verbose:gc -Xms80M -Xmx80M -Xmn10M
 * // -XX:+PrintGCDetails -XX:SurvivorRatio=8
 */
public class ReuseVariableSlot {
    public static void main(String[] args) {
        {
            byte[] placeholder = new byte[64 * 1024 * 1024];
        }
         int a = 0;
        System.gc();
    }
} 
 
 result: 
 [0.026s][info][gc] Using G1
[0.145s][info][gc] GC(0) Pause Initial Mark (G1 Humongous Allocation) 2M->1M(80M) 1.527ms
[0.145s][info][gc] GC(1) Concurrent Cycle
[0.170s][info][gc] GC(1) Pause Remark 66M->66M(80M) 0.789ms
[0.170s][info][gc] GC(1) Pause Cleanup 66M->66M(80M) 0.351ms
[0.170s][info][gc] GC(1) Concurrent Cycle 25.457ms
[0.174s][info][gc] GC(2) Pause Full (System.gc()) 66M->1M(80M) 3.362ms 

```

我们发现加上 int a = 0; 之后，发现内存被回收了 （[info][gc] GC(2) Pause Full (System.gc()) 66M->1M(80M) 3.362ms ）



##### 代码总结：

上面的代码 ，**placeholder 能否被回收** 的根本原因是：  

- 局部变量表的变量槽 是否还存有 关于placeholder数组对象的 **引用**
- 第一次修改中（加花括号），代码虽然 已经离开了placeholder的作用域，但在之后，在没有发生过任何对局部变量表的读写 操作，placeholder 原本所占用的变量槽  还没有被其他变量所复用，所以作为 GC ROOTs 一部分的局部变量表仍然保持着对它的关联。这种关联没有被及时打断。再绝大部分情况下影响很轻微。 但如果遇到 一个方法，其后面的代码有一些耗时很长的操作，而前面又定义了占用大部分内存但实际已经不会再使用的变量， 手动 将其设置为 null值 （用来代替 int a=0，把变量对应的局部变量槽清空）便不见得是一个绝对无意义的操作 。这是**奇技淫巧** ，并不是很认同。
- 这种操作可以作为一种在极特殊情形（对象占用内存大、此方法的栈帧长时间不能被回收、方法调用次数达不到即时编译器的编译条件）下的“奇技”来使用。Java语言的一本非常著名的书籍《Practical Java》中将把“不使用的对象应手动赋值为null”作为一条推荐的编码规则（笔者并不认同这条规则），但是并没有解释具体原因
- 这是**奇技淫巧** ，并不是很认同。因为：
  - 从编码角度讲，以恰当的变量作用域来控制变量回收时间才是最优雅的解决方法
  - 更关键的是，从执行角度来讲，使用赋null操作来优化内存回收是建立在对字节码执行引擎概念模型的理解之上的，概念模型与实际执行过程是外部看起来等效，内部看上去则可以完全不同。当虚拟机使用解释器执行时，通常与概念模型还会比较接近，但经过即时编译器施加了各种编译优化措施以后，两者的差异就会非常大，只保证程序执行的结果与概念一致。在实际情况中，即时编译才是虚拟机执行代码的主要方式，赋null值的操作在经过即时编译优化后几乎是一定会被当作无效操作消除掉的，这时候将变量设置为null就是毫无意义的行为。字节码被即时编译为本地代码后，对GC Roots的枚举也与解释执行时期有显著差别，以前面的例子来看，经过第一次修改的代码（第二个方法）在经过即时编译后，System.gc()执行时就可以正确地回收内存，根本无须写成代码（第三个方法）的样子。



##### 局部变量表不存在准备阶段

- 局部变量表不像类变量那样存在“准备阶段”，
- 之前我们在《虚拟机的类加载过程》学过，类的字段变量 有2次赋初始值的过程，  一次是在 准备阶段，赋予系统初始值；另外一次是在 初始化阶段，赋予程序员在代码中定义的初始值。 因此 即使在初始化阶段 程序员没有为类变量赋值也没有关系，类变量仍然具有一个决定的初始值（零值，默认值），不会产生歧义。

- 但是 局部变量就不一样 了，如果一个局部变量定义了，但没有赋初始值，那它是完全不能使用的。
- 所以不要认为 Java中任何情况都存在 诸如 整型变量默认为0、布尔型变量默认为 false 等这样的默认值规则。
- 如果局部变量不赋初始值，编译期在编译期间能检查到并提示出来，ide工具是会报错的。
- 即使编译能通过或者 通过手动的方式生成字节码，在《虚拟机的类加载过程》的字节码验证阶段，也会被虚拟机发现，从而导致 类加载失败。

```java
public static void main(String[] args) {
        int a; //
        System.out.println(a);

    }

result:  编译期报错：  可能尚未初始化变量a
```



### 操作数栈（Operand Stack）

- 操作数栈 （Operand Stack）也常被称为操作栈，它是一个**先入后出** (Last In First Out,LIFO） 栈。 

- 同局部变量表一样， 操作数栈的最大深度也在编译的时候被写入到 Code 属性的 **max_stacks** 数据项之中。

- 操作数栈的每一个元素 都可以是包括long 和 double 在内的**任意java数据类型** 。32位 数据类型 所占的栈容量为 1，64 位数据类型所占的栈容量为  2。

- javac 编译器的数据流分析工作保证了 在方法执行的任何时候，操作数栈的深度 都不会超过 在 max_stacks 数据项设定的最大值。



##### 方法执行时的操作栈动作

- 当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的。  在方法的执行过程中，会有各种**字节码指令** 往  **操作数栈** 中写入和提取 内容， 也就是出栈和入栈操作。

 譬如 在做算术运算的时候 ，通过将运算涉及的操作数栈 压入栈顶后 ，然后 调用运算指令来执行。

又譬如 在调用其他方法的时候 ，通过操作数栈 来进行方法参数的传递。

举个例子， 例如整数加法 的字节码 iadd, 这条指令在运行的时候 ，要求操作数栈中最接近 栈顶的2个元素 已经存入了2个int 型的数值，当执行 iadd这个指令时候，会把这2 个int 值出栈 并相加，然后将 相加的结果重新入栈 。

##### 字节码指令严格匹配操作栈元素数据类型

操作数栈中元素的数据类型必须 与字节码指令的序列严格匹配，在编译程序代码的时候，编译期必须严格保证这一点，在类校验阶段的数据流分析中还要 再次验证这一点。

例如： iadd指令只能 用于整型数据的加法，该指令在执行时，最接近栈顶的2个元素的数据类型 必须为int 型，不能出现一个long 和一个float 使用iadd指令相加的 情况。

##### 栈帧的优化

- Java虚拟机的**解释执行引擎** 被称为“**基于栈的执行引擎**”, 里面的栈 就是操作数栈。后续还会对基于栈的代码执行过程进行更详细的讲解。介绍它与更常见的**基于寄存器的执行引擎**有那些差别

- 在概念 模型中，两个不同栈帧（Stack Frame）作为不同方法的虚拟机栈 元素，是**完全互相独立**的。  但是在大多数虚拟机的实现中，会对这块进行一些**优化处理** ，令2个 不同方法的栈帧出现一部分 **重叠**。让下面的栈帧的部分操作数栈 与 上面栈帧的部分局部变量表 重叠在一起，这样做不仅节约了 一些空间，更重要的是 在进行方法调用的时候 就可以直接共用 一部分数据，无需进行**额外的 参数 复制传递**了 。重叠的过程如下图所示： 

![](/uploads/jvm/10StackFrame/02-StackFrameOverlap.png)



### 动态连接（Dynamic Linking）

每个栈帧 都包含一个指向**运行时常量池** 中**该栈帧所属方法的引用**，持有这个引用 是为了支持方法调用过程中的**动态连接**。

通过之前文章，我们知道 Class文件 的常量池中存有 大量的符号引用，字节码中的**方法调用指令** 就以 常量池里**指向方法的符号连接** 作为参数。

- 这些符号引用一部分将在 **类加载阶段**或者**第一次使用的时候**就被转化为**直接引用**，这种转化称为  **静态解析**
- 另外一部分将在**每一次运行期间**都**转化 为直接引用**，这部分就称为 **动态连接**。后续后详细讲解



### 方法返回地址

当一个方法开始执行后，只有 2种 方式退出这个方法。

- 第一种退出方式是执行引擎遇到任意一个方法**返回的字节码指令**，这时候 可能会有返回值 传递给上层方法调用者 （调用当前方法的方法称为 **主调方法** 或者 “**调用者**”）， 方法是否有返回值 以及返回值的类型 将根据 何种方法返回指令来决定，这种退出方式 称为 “**正常调用完成（Normal Method Invocation Completion）**”
- 另外一种退出方式 就是方法执行过程中 遇到了异常，并且这个异常没有在方法体内得到妥善处理。  无论是Java虚拟机内部的异常，还是代码中**athrow字指令** 产生的异常，只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种退出方法的方式 称为 “**异常调用完成(Abrupt Method Invocation Completion)**”  。一个方法使用异常完成出口的方式退出，是不会给他的上层调用者 提供任何返回值的。



无论采用何种退出方式，在方法退出之后，都必须**返回最初方法被调用的位置**，程序才能继续执行。

方法返回时 可能需要在栈帧中 保存一些信息，用来帮助**恢复它的上层主调方法的执行状态** 。 

一般来说，方法正常退出时，**主调方法的PC计数器的值 就可以作为返回地址**，栈帧中很可能 会保存这个计数器值。

而异常退出的时候，返回地址就是 通过异常处理器 来确定的，栈帧中 一般不会保存这部分信息 （主调方法的PC计数器值）

##### 方法退出的过程

方法退出的过程实际上等同于**把 当前栈帧出栈**，因此退出时可能执行的操作有：

- 恢复上层主调方法的局部变量表和操作数栈，
- 把返回值（如果有的话） 压入 调用者（主调方法对应的）栈帧的操作数栈中，
- 调整PC计数器值 以指向方法调用指令后面的 的一条指令等。

这些都是 基于概念模型的讨论，只有具体到某一款 java虚拟机实现，会执行哪些操作才能确定下来



### 附加信息

《Java虚拟机规范》允许虚拟机实现增加一些规范里没有描述的信息到栈帧之中，例如与调试、性能收集相关的信息，这部分信息完全取决于具体的虚拟机实现。

在讨论概念时候，一般会把 **动态连接，方法返回地址 与 其他附加信息**全部归为一类，称为**栈帧信息** 。



## 方法调用详解（TODO）

方法调用并不等同于方法中的代码被执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还未涉及方法内部的具体执行过程。

在程序执行时，进行方法调用是最普遍最频繁的操作之一，Class文件的编译过程中 不包含传统程序语言编译的连接步骤，一切方法调用在Class文件 里面存储的都是 符号引用，而不是方法在实际运行时内存布局中的入口地址（也就是之前说的直接引用）。 这个特性给了Java带来了更强大的动态扩展能力，也使得Java方法调用过程变得相对复杂，某些调用需要在类加载期间，甚至到运行期间才能确定目标方法的直接引用。

### 解析

所有方法调用的目标方法在Class 文件里面都是一个 常量池的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用，这种解析能够成立的前提是:  方法在程序真正执行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。 换句话说，调用目标在程序代码写好、编译器进行编译那一刻就已经确定下来。这类方法的调用被称为**解析**（**Resolution**）

#### 编译期可知，运行期不可变

在Java语言中，符合“编译期可知，运行期不可变”这个要求的方法，主要有 **静态方法** 和**私有方法**两大类，前者与类型直接关联，后者在外部不可访问，这两种方法各自的特点决定了他们都不可能通过继承或 别的方式重写出其他版本，因为它们都适合在类解析阶段进行解析。

#### 方法调用字节码指令

调用不同类型的方法，字节码指令集里面设计了不同的指令。在Java虚拟机支持 以下5条方法调用字节码指令，分别是：

1. invokestatic ： 用于调用静态方法
2. invokespecial：  用于调用实例构造器<init>()方法、私有方法和父类中的方法
3. invokevirtual :  用于调用所有的虚方法
4. invokeinterface: 用于调用接口方法，会在运行时在确定一个实现该接口的对象
5. invokedynamic：  现在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法。

#### 虚方法与非虚方法

- 前面的1-4条调用指令，分配逻辑都固化在Java虚拟机内部，而invokedynamic指令的分派逻辑是由用户设定的引导方法来决定的。
- 只要能被invokespecial 和 invokestatic指令调用的方法，都可以在解析阶段中确定唯一的调用版本 ，Java语言里符合这个条件的方法 有： 

**静态方法、私有方法、实例构造器、父类方法**4种，再加上被 final修饰的方法（尽管它用invokevirtual）指令调用，这5种方法调用会再类加载的时候就可以把符号引用解析为该方法的直接引用。这些方法统称为“**非虚方法**”（**Non-Virtual Method**），与之相反的，其他方法就被称为 **虚方法（Virtual Method）**



方法静态解析演示：

演示了一种常见的解析调用的例子，该样例中，**静态方法sayHello 只可能属于类型 StaticResolution ，没有任何途径可以覆盖或隐藏这个方法**：



```java
/**
 * 方法静态解析演示
 *
 * @author zzm
 */
public class StaticResolution {

    public static void sayHello() {
        System.out.println("hello world");
    }

    public static void main(String[] args) {
        StaticResolution.sayHello();
    }

}
```



使用 javap -verbose StaticResolution.class   得到下面结果：

使用javap命令查看这段程序对应的字节码，会发现的确是通过**invokestatic**命令来调用sayHello()方法，而且其调用的方法版本已经在编译时就明确以**常量池项的形式**固化在字节码指令的参数之中（ #5 = Methodref ）

```java
 Last modified 2021-1-18; size 672 bytes
  MD5 checksum 4041d61106416a1bb60c338694bf8d00
  Compiled from "StaticResolution.java"
public class org.fenixsoft.jvm.chapter8.StaticResolution
  minor version: 0
  major version: 53
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #7.#22         // java/lang/Object."<init>":()V
   #2 = Fieldref           #23.#24        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #25            // hello world
   #4 = Methodref          #26.#27        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Methodref          #6.#28         // org/fenixsoft/jvm/chapter8/StaticResolution.sayHello:()V
   #6 = Class              #29            // org/fenixsoft/jvm/chapter8/StaticResolution
   #7 = Class              #30            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               LocalVariableTable
  #13 = Utf8               this
  #14 = Utf8               Lorg/fenixsoft/jvm/chapter8/StaticResolution;
  #15 = Utf8               sayHello
  #16 = Utf8               main
  #17 = Utf8               ([Ljava/lang/String;)V
  #18 = Utf8               args
  #19 = Utf8               [Ljava/lang/String;
  #20 = Utf8               SourceFile
  #21 = Utf8               StaticResolution.java
  #22 = NameAndType        #8:#9          // "<init>":()V
  #23 = Class              #31            // java/lang/System
  #24 = NameAndType        #32:#33        // out:Ljava/io/PrintStream;
  #25 = Utf8               hello world
  #26 = Class              #34            // java/io/PrintStream
  #27 = NameAndType        #35:#36        // println:(Ljava/lang/String;)V
  #28 = NameAndType        #15:#9         // sayHello:()V
  #29 = Utf8               org/fenixsoft/jvm/chapter8/StaticResolution
  #30 = Utf8               java/lang/Object
  #31 = Utf8               java/lang/System
  #32 = Utf8               out
  #33 = Utf8               Ljava/io/PrintStream;
  #34 = Utf8               java/io/PrintStream
  #35 = Utf8               println
  #36 = Utf8               (Ljava/lang/String;)V
{
  public org.fenixsoft.jvm.chapter8.StaticResolution();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 8: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lorg/fenixsoft/jvm/chapter8/StaticResolution;

  public static void sayHello();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello world
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 11: 0
        line 12: 8

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=0, locals=1, args_size=1
         0: invokestatic  #5                  // Method sayHello:()V
         3: return
      LineNumberTable:
        line 15: 0
        line 16: 3
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       4     0  args   [Ljava/lang/String;
}
SourceFile: "StaticResolution.java"

```

Java中的非虚方法除了使用  **invokespecial、invokestatic** 调用的方法之外，还有一种就是 **被final 修饰的实例方法**。历史设计的原因，final 方法使用的是 **invokevirtual** 指令调用的，但是因为它无法被覆盖，没有其他版本的可能，所以也无须 对方法接收者 进行多态选择，又或者说 多态选择的结果肯定是唯一的。

再《Java语言规范》中明确定义了被 final修饰的方法是一种**非虚方法**.



#### 解析调用：

解析调用一定是一个静态过程，在编译期间就完全确定，在类加载的解析阶段就会把涉及的符号引用全部 转变为 明确的直接引用，不必延迟到运行期再去完成。

#### 分派调用：

另一种主要的方法调用形式：  **分派（Dispatch）**调用则复杂许多，它可能是静态的也可能是动态的，按照分派依据 的宗量数可分为 **单分派** 和 **多分派**。 这两类分派两两组合就构成了静态单分派、静态多分派、动态单分派、动态多分派 4种分派组合情况 

### 分派

As we all know, Java 是一门面向对象的程序语言，因为java具备面向对象的 3个基本特征：  **继承、封装 和多态。**

#### 静态分派

 **“分派”（Dispatch）** 这个词本身就具有动态性，一般不应用在静态语境中。这部分原本在 英文原版的《Java虚拟机规范》和 《Java语言规范》里的说法都是:  

"Method Overload Resolution 方法重载解析" ,但部分外文资料和国内翻译的许多中文资料都将这种行为 称为 “静态分派”。

为了解释静态分派和重载（Overload），笔者准备了一段经常出现在面试题中的程序代码，读者不妨先看一遍，想一下程序的输出结果是什么。后面的话题将围绕这个类的方法来编写重载代码，以分析虚拟机和编译器确定方法版本的过程。程序如代码清单8-6所示。

```java
/**
 * 方法静态分派演示
 * @author zzm
 */
public class StaticDispatch {

    static abstract class Human {
    }

    static class Man extends Human {
    }

    static class Woman extends Human {
    }

    public void sayHello(Human guy) {
        System.out.println("hello,guy!");
    }

    public void sayHello(Man guy) {
        System.out.println("hello,gentleman!");
    }

    public void sayHello(Woman guy) {
        System.out.println("hello,lady!");
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        StaticDispatch sr = new StaticDispatch();
        sr.sayHello(man);
        sr.sayHello(woman);
    }
}


运行结果：
    hello,guy!
hello,guy!

```



为什么虚拟机会选择执行参数类型为Human的重载版本呢？在解决这个问题之前，我们先通过如下代码来定义两个关键概念：

```java
Human man = new Man();
```

- 我们把上面代码中的 “Human” 称为 变量的 **“静态类型”（static Type）**，或者叫 **“外观类型”（Apparent Type）**,
- "Man" 则被称为变量的 **“实际类型”（Actual Type）**或者叫做 **“运行时类型”（Runtime Type）**.
- 静态类型和实际类型在程序中都可能会发生变化，区别是 静态类型的变化仅仅在使用时发生， 变量本身的静态类型不会被改变，并且最终的静态类型在编译期可知的；而 实际类型变化的结果在运行期才可确定，编译期在编译程序的时候并不知道一个对象的实际类型时什么

```java
// 实际类型变化
Human human = (new Random()).nextBoolean() ? new Man() : new Woman();
// 静态类型变化
sr.sayHello((Man) human)
sr.sayHello((Woman) human)

```

- 对象human 的实际类型是可变的，编译期间 它完全是个 “薛定谔的人”，到底是 Man 还是woman，必须等到程序运行到这行的时候才能确定。

- 而human的静态类型是Human，也可以在使用时（如sayHello()方法中的强制转型）临时改变这个类型，但这个改变仍是在编译期是可知的，两次sayHello()
  方法的调用，在编译期完全可以明确转型的是Man还是Woman。

main()里面的两次sayHello()方法调用，在方法接收者已经确定是对象“sr”的前提下，使用哪个重载版本，就完全**取决于传入参数的数量和数据类型**。

代码中故意**定义了两个静态类型相同**，而**实际类型不同的变量**，但虚拟机（或者准确地说是编译器）**在重载时是通过参数的静态类型**而**不是实际类型**作为判定依据的。由于**静态类型在编译期可知**，所以**在编译阶段**，Javac编译器就**根据参数的静态类型决定了会使用哪个重载版本**，因此选择了**sayHello(Human)作为调用目标**，并把**这个方法的符号引用写到main()方法里的两条invokevirtual指令的参数中。**



```java
public class org.fenixsoft.jvm.chapter8.StaticDispatch
  minor version: 0
  major version: 53
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #14.#44        // java/lang/Object."<init>":()V
   #2 = Fieldref           #45.#46        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #47            // hello,guy!
   #4 = Methodref          #48.#49        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = String             #50            // hello,gentleman!
   #6 = String             #51            // hello,lady!
   #7 = Class              #52            // org/fenixsoft/jvm/chapter8/StaticDispatch$Man
   #8 = Methodref          #7.#44         // org/fenixsoft/jvm/chapter8/StaticDispatch$Man."<init>":()V
   #9 = Class              #53            // org/fenixsoft/jvm/chapter8/StaticDispatch$Woman
  #10 = Methodref          #9.#44         // org/fenixsoft/jvm/chapter8/StaticDispatch$Woman."<init>":()V
  #11 = Class              #54            // org/fenixsoft/jvm/chapter8/StaticDispatch
  #12 = Methodref          #11.#44        // org/fenixsoft/jvm/chapter8/StaticDispatch."<init>":()V
  #13 = Methodref          #11.#55        // org/fenixsoft/jvm/chapter8/StaticDispatch.sayHello:(Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
  #14 = Class              #56            // java/lang/Object
  #15 = Utf8               Woman
  #16 = Utf8               InnerClasses
  #17 = Utf8               Man
  #18 = Class              #57            // org/fenixsoft/jvm/chapter8/StaticDispatch$Human
  #19 = Utf8               Human
  #20 = Utf8               <init>
  #21 = Utf8               ()V
  #22 = Utf8               Code
  #23 = Utf8               LineNumberTable
  #24 = Utf8               LocalVariableTable
  #25 = Utf8               this
  #26 = Utf8               Lorg/fenixsoft/jvm/chapter8/StaticDispatch;
  #27 = Utf8               sayHello
  #28 = Utf8               (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
  #29 = Utf8               guy
  #30 = Utf8               Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;
  #31 = Utf8               (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Man;)V
  #32 = Utf8               Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Man;
  #33 = Utf8               (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Woman;)V
  #34 = Utf8               Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Woman;
  #35 = Utf8               main
  #36 = Utf8               ([Ljava/lang/String;)V
  #37 = Utf8               args
  #38 = Utf8               [Ljava/lang/String;
  #39 = Utf8               man
  #40 = Utf8               woman
  #41 = Utf8               sr
  #42 = Utf8               SourceFile
  #43 = Utf8               StaticDispatch.java
  #44 = NameAndType        #20:#21        // "<init>":()V
  #45 = Class              #58            // java/lang/System
  #46 = NameAndType        #59:#60        // out:Ljava/io/PrintStream;
  #47 = Utf8               hello,guy!
  #48 = Class              #61            // java/io/PrintStream
  #49 = NameAndType        #62:#63        // println:(Ljava/lang/String;)V
  #50 = Utf8               hello,gentleman!
  #51 = Utf8               hello,lady!
  #52 = Utf8               org/fenixsoft/jvm/chapter8/StaticDispatch$Man
  #53 = Utf8               org/fenixsoft/jvm/chapter8/StaticDispatch$Woman
  #54 = Utf8               org/fenixsoft/jvm/chapter8/StaticDispatch
  #55 = NameAndType        #27:#28        // sayHello:(Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
  #56 = Utf8               java/lang/Object
  #57 = Utf8               org/fenixsoft/jvm/chapter8/StaticDispatch$Human
  #58 = Utf8               java/lang/System
  #59 = Utf8               out
  #60 = Utf8               Ljava/io/PrintStream;
  #61 = Utf8               java/io/PrintStream
  #62 = Utf8               println
  #63 = Utf8               (Ljava/lang/String;)V
{
  public org.fenixsoft.jvm.chapter8.StaticDispatch();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lorg/fenixsoft/jvm/chapter8/StaticDispatch;

  public void sayHello(org.fenixsoft.jvm.chapter8.StaticDispatch$Human);
    descriptor: (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String hello,guy!
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 19: 0
        line 20: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lorg/fenixsoft/jvm/chapter8/StaticDispatch;
            0       9     1   guy   Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;

  public void sayHello(org.fenixsoft.jvm.chapter8.StaticDispatch$Man);
    descriptor: (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Man;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #5                  // String hello,gentleman!
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 23: 0
        line 24: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lorg/fenixsoft/jvm/chapter8/StaticDispatch;
            0       9     1   guy   Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Man;

  public void sayHello(org.fenixsoft.jvm.chapter8.StaticDispatch$Woman);
    descriptor: (Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Woman;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String hello,lady!
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 27: 0
        line 28: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lorg/fenixsoft/jvm/chapter8/StaticDispatch;
            0       9     1   guy   Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Woman;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #7                  // class org/fenixsoft/jvm/chapter8/StaticDispatch$Man
         3: dup
         4: invokespecial #8                  // Method org/fenixsoft/jvm/chapter8/StaticDispatch$Man."<init>":()V
         7: astore_1
         8: new           #9                  // class org/fenixsoft/jvm/chapter8/StaticDispatch$Woman
        11: dup
        12: invokespecial #10                 // Method org/fenixsoft/jvm/chapter8/StaticDispatch$Woman."<init>":()V
        15: astore_2
        16: new           #11                 // class org/fenixsoft/jvm/chapter8/StaticDispatch
        19: dup
        20: invokespecial #12                 // Method "<init>":()V
        23: astore_3
        24: aload_3
        25: aload_1
        26: invokevirtual #13                 //这里： Method sayHello:(Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
        29: aload_3
        30: aload_2
        31: invokevirtual #13                 //这里： Method sayHello:(Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;)V
        34: return
      LineNumberTable:
        line 31: 0
        line 32: 8
        line 33: 16
        line 34: 24
        line 35: 29
        line 36: 34
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      35     0  args   [Ljava/lang/String;
            8      27     1   man   Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;
           16      19     2 woman   Lorg/fenixsoft/jvm/chapter8/StaticDispatch$Human;
           24      11     3    sr   Lorg/fenixsoft/jvm/chapter8/StaticDispatch;
}
SourceFile: "StaticDispatch.java"
InnerClasses:
     static #15= #9 of #11; //Woman=class org/fenixsoft/jvm/chapter8/StaticDispatch$Woman of class org/fenixsoft/jvm/chapter8/StaticDispatch
     static #17= #7 of #11; //Man=class org/fenixsoft/jvm/chapter8/StaticDispatch$Man of class org/fenixsoft/jvm/chapter8/StaticDispatch
     static abstract #19= #18 of #11; //Human=class org/fenixsoft/jvm/chapter8/StaticDispatch$Human of class org/fenixsoft/jvm/chapter8/StaticDispatch
```



所有依赖静态类型来决定执行版本的分派动作，都被称为 **静态分派**。 静态分配的最典型应用表现就是 **方法重载**。 静态分派 发生在编译阶段，因此确定静态分派的动作实际上并不是由虚拟机来执行的 ，这点也是为什么一些资料选择把它归入 为 “解析”而不是 “分派”的原因。



##### 重载方法匹配优先级

需要注意Javac编译器虽然能确定出方法的重载版本，但在很多情况下这个重载版本并不是“唯一”的，往往只能确定一个“相对更合适的”版本。这种模糊的结论在由0和1构成的计算机世界中算是个比较稀罕的事件，产生这种模糊结论的主要原因是字面量天生的模糊性，它不需要定义，所以字面量就没有显式的静态类型，它的静态类型只能通过语言、语法的规则去理解和推断。代码清单8-7演示了何谓“更加合适的”版本。

代码清单8-7　重载方法匹配优先级

```java
/**
 * @author zzm
 */
public class Overload {

    public static void sayHello(Object arg) {
        System.out.println("hello Object");
    }

    public static void sayHello(int arg) {
        System.out.println("hello int");
    }

    public static void sayHello(long arg) {
        System.out.println("hello long");
    }

    public static void sayHello(Character arg) {
        System.out.println("hello Character");
    }

    public static void sayHello(char arg) {
        System.out.println("hello char");
    }

    public static void sayHello(char... arg) {
        System.out.println("hello char ...");
    }

    public static void sayHello(Serializable arg) {
        System.out.println("hello Serializable");
    }

    public static void main(String[] args) {
        sayHello('a');
    }
}
```

- 第一次运行结果： **hello char**，这很好理解，'a'是一个char类型的数据，自然会寻找参数类型为char的重载方法，
- 如果注释掉 sayHello(char arg)方法，那输出会变为：**hello int** ，这时发生了一次自动类型转换，'a'除了可以代表一个字符串，还可以代表数字97（字符'a'的
  Unicode数值为十进制数字97），因此参数类型为int的重载也是合适的

- 我们继续注释掉sayHello(int arg)方法，那输出会变为： **hello long** .这时发生了两次自动类型转换，'a'转型为整数97之后，进一步转型为长整数97L，匹配了参数类型为long的重载。笔者在代码中没有写其他的类型如float、double等的重载，不过实际上自动转型还能继续发生多次，按照**char>int>long>float>double的顺序转型进行匹配**，但**不会匹配到byte和short类型的重载**，因为**char到byte或short的转型是不安全的**。
- 继续注释掉sayHello(long arg)方法，那输出会变为 : **hello Character**  ，这时发生了一次自动装箱，'a'被包装为它的封装类型java.lang.Character，所以匹配到了参数类型为Character的重载
- 继续注释掉sayHello(Character arg)方法，那输出会变为： **hello Serializable**,  出现hello Serializable，是因为java.lang.Serializable是java.lang.Character类实现的一个接口，当自动装箱之后发现还是找不到装箱类，但是找到了装箱类所实现的接口类型，所以紧接着又发生一次自动转型。char可以转型成int，
  但是Character是绝对不会转型为Integer的，它只能安全地转型为它实现的接口或父类。Character还实现了另外一个接口java.lang.Comparable<Character>，如果同时出现两个参数分别为Serializable和Comparable<Character>的重载方法，那它们在此时的优先级是一样的。编译器无法确定要自动转型为哪种类型，会提示“类型模糊”（Type Ambiguous），并拒绝编译。程序必须在调用时显式地指定字面量的静态类型，如：sayHello((Comparable<Character>)'a')，才能编译通过
- 继续注释掉sayHello(Serializable arg)方法，输出会变为： **hello Object** ，这时是char装箱后转型为父类了，如果有多个父类，那将在继承关系中从下往上开始搜索，越接上层的优先级越低。即使方法调用传入的参数值为null时，这个规则仍然适用。
- 我们把sayHello(Objectarg)也注释掉，输出将会变为： **hello char ...**,7个重载方法已经被注释得只剩1个了，可见变长参数的重载优先级是最低的，这时候字符'a'被当作了一个char[]数组的元素。笔者使用的是char类型的变长参数，读者在验证时还可以选择int类型、Character类型、Object类型等的变长参数重载来把上面的过程重新折腾一遍。但是要注意的是，有一些在单个参数中能成立的自动转型，如char转型为int，在变长参数中是不成立的[2]。

另外还有一点读者可能比较容易混淆：笔者讲述的**解析与分派这两者之间的关系并不是二选一的排他关系**，它们是在**不同层次上去筛选、确定目标方法**的过程。例如前面说过静态方法会在编译期确定、在类加载期就进行解析，而静态方法显然也是可以拥有重载版本的，**选择重载版本的过程也是通过静态分派**完成的

 

- 可以这么理解吗： **overload重载是gen据静态类型来决定的（静态分派），  override是根据 实际类型来决定的（动态分派）。**



#### 动态分派

看一下Java语言里动态分派的实现过程，它与Java语言多态性的另外一个重要体现[3]——重写（Override）有着很密切的关联。我们还是用前面的Man和Woman一起 sayHello的例子来讲解动态分派，请看代码清单8-8中所示的代码。

代码清单8-8　方法动态分派演示

```java
/**
 * 方法动态分派演示
 * @author zzm
 */
public class DynamicDispatch {

    static abstract class Human {
        protected abstract void sayHello();
    }

    static class Man extends Human {
        @Override
        protected void sayHello() {
            System.out.println("man say hello");
        }
    }

    static class Woman extends Human {
        @Override
        protected void sayHello() {
            System.out.println("woman say hello");
        }
    }

    public static void main(String[] args) {
        Human man = new Man();
        Human woman = new Woman();
        man.sayHello();
        woman.sayHello();
        man = new Woman();
        man.sayHello();
    }
}


运行结果：
    man say hello
woman say hello
woman say hello
```

##### Java虚拟机是如何判断应该调用哪个方法

- 可以这么理解吗： **overload重载是gen据静态类型来决定的（静态分派），  override是根据 实际类型来决定的（动态分派）。**

显然这里选择调用的方法版本是不可能再根据静态类型来决定的，因为**静态类型同样都是Human的两个变量**man和woman在调用sayHello()方法时产生了不同的行为，甚至变量man在两次调用中还执行了两个不同的方法。导致这个现象的原因很明显，是因为这两个变量的实际类型不同，Java虚拟机是如何根据实际类型来分派方法执行版本的呢？我们使用javap命令输出这段代码的字节码，尝试从中寻找答案，输出结果如代码清单8-9所示。

```bash
public static void main(java.lang.String[]);
	Code:
        Stack=2, Locals=3, Args_size=1
        0: new #16; //class org/fenixsoft/polymorphic/DynamicDispatch$Man
        3: dup
        4: invokespecial #18; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Man."<init>":()V
        7: astore_1
        8: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
        11: dup
        12: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
        15: astore_2
        16: aload_1
        17: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        20: aload_2
        21: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        24: new #19; //class org/fenixsoft/polymorphic/DynamicDispatch$Woman
        27: dup
        28: invokespecial #21; //Method org/fenixsoft/polymorphic/DynamicDispatch$Woman."<init>":()V
        31: astore_1
        32: aload_1
        33: invokevirtual #22; //Method org/fenixsoft/polymorphic/Dynamic Dispatch$Human.sayHello:()V
        36: return
```

0～15行的字节码是准备动作，作用是建立man和woman的内存空间、调用Man和Woman类型的实例构造器，将这两个实例的引用存放在第1、2个局部变量表的变量槽中，这些动作实际对应了Java源码中的这两行： 

```java
Human man = new Man();
Human woman = new Woman();
```

接下来的16～21行是关键部分，16和20行的**aload指令**分别把刚刚创建的两个对象的引用压到栈顶，这两个对象是将要执行的sayHello()方法的所有者，称为接收者（Receiver）；

17和21行是方法调用指令，这两条调用指令单从字节码角度来看，无论是指令（都是invokevirtual）还是参数（都是常量池中第22项的常量，注释显示了这个常量是Human.sayHello()的符号引用）都完全一样，但是这两句指令最终执行的目标方法并不相同。那看来解决问题的关键还必须从invokevirtual指令本身入手，要弄清楚它是如何确定调用方法版本、如何实现多态查找来着手分析才行。根据《Java虚拟机规范》，invokevirtual指令的运行时解析过程[4]大致分为以下几步：

1. **找到操作数栈顶的第一个元素所指向的对象的实际类型，记作C**。
2. 如果在类型C中找到与常量中的描述符和简单名称都相符的方法，则进行访问权限校验，**如果通过则返回这个方法的直接引用**，查找过程结束；不通过则返回java.lang.**IllegalAccessError**异常。
3. 否则，按照**继承关系从下往上依次**对C的各个父类进行第二步的**搜索和验证**过程。
4. 如果始终没有找到合适的方法，则抛出java.lang.**AbstractMethodError**异常。

正是因为invokevirtual指令执行的**第一步就是在运行期确定接收者的实际类型**，所以两次调用中的invokevirtual指令并**不是把常量池中方法的符号引用解析到直接引用上**就结束了，还会**根据方法接收者的实际类型来选择方法版本**，这个过程就是**Java语言中方法重写的本质**。我们把这种在运行期根据实际类型确定方法执行版本的分派过程称为**动态分派。**

既然这种多态性的**根源在于虚方法调用指令invokevirtual的执行逻辑**，那自然我们得出的结论就**只会对方法有效，对字段是无效的，因为字段不使用这条指令**。事实上，在Java里面只有虚方法存在，字段永远不可能是虚的，

换句话说，字段永远不参与多态，哪个类的方法访问某个名字的字段时，该名字指的就是这个类能看到的那个字段。**当子类声明了与父类同名的字段时**，虽然在子类的内存中两个字段都会存在，但是**子类的字段会遮蔽父类的同名字段**。

为了加深理解，又编撰了一份“劣质面试题式”的代码片段，请阅读代码清单8-10，思考运行后会输出什么结果。

代码清单8-10　字段没有多态性

```java
/**
 * 字段不参与多态
 * @author zzm
 */
public class FieldHasNoPolymorphic {

    static class Father {
        public int money = 1;

        public Father() {
            money = 2;
            showMeTheMoney();
        }

        public void showMeTheMoney() {
            System.out.println("I am Father, i have $" + money);
        }
    }

    static class Son extends Father {
        public int money = 3;

        public Son() {
            money = 4;
            showMeTheMoney();
        }

        public void showMeTheMoney() {
            System.out.println("I am Son,  i have $" + money);
        }
    }

    public static void main(String[] args) {
        Father gay = new Son();
        System.out.println("This gay has $" + gay.money);
    }
}

运行结果为: 
I am Son,  i have $0
I am Son,  i have $4
This gay has $2

```

输出两句都是“I am Son”，

- 这是因为Son类在创建的时候，首先隐式调用了**Father的构造函数**，而Father构造函数中**对showMeTheMoney()的调用是一次虚方法调用**，实际执行的版本是Son::showMeTheMoney()方法，所以输出的是“I am Son”，这点经过前面的分析相信读者是没有疑问的了。
- 而这时候虽然父类的money字段已经被初始化成2了，但Son::showMeTheMoney()方法中访问的却是**子类的money字段**，这时候结果自然还是0，因为它要到子类的构造函数执行时才会被初始化。
- main()的最后一句**通过静态类型访问到了父类中的money**，输出了2。

**todo  消化一下**



#### 单分派与多分派

##### 宗量

方法的接收者与方法的参数统称为方法的**宗量**

根据分派基于多少种宗量，可以将分派划分为**单分派**和**多分派**两种。

单分派 是根据**一个宗量对目标进行选择**，

多分派则是根据**多于一个宗量对目标方法进行选择。**



代码清单：  单分派和多分派

```java
/**
 * 单分派、多分派演示
 * @author zzm
 */
public class Dispatch {

    static class QQ {}

    static class _360 {}

    public static class Father {
        public void hardChoice(QQ arg) {
            System.out.println("father choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("father choose 360");
        }
    }

    public static class Son extends Father {
        public void hardChoice(QQ arg) {
            System.out.println("son choose qq");
        }

        public void hardChoice(_360 arg) {
            System.out.println("son choose 360");
        }
    }

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son();
        father.hardChoice(new _360());
        son.hardChoice(new QQ());
    }
}


运行结果： 
    father choose 360
	son choose qq
```

- main（）里调用了两次hardChoice 方法，这两次hardChoice 方法的选择结果在程序输出 中已经显示得很清楚了。 
- 我们关注得首先是 编译阶段中编译 器的选择过程，也就是静态分派的过程。
- 选择目标的依据有两点：
  - 一个是静态类型是 Father 还是SON
  - 二是方法参数是QQ还是360

这次选择结果的最终产物是产生了2条invokevirtual 指令，两条指令的参数 分别是常量池中指向 Father:: hardChoice(360)  以及 Father:: hardChoice(QQ)方法的符号引用

- 因为是根据两个宗量进行选择，所以Java语言的静态分派属于多分派类型



##### 动态分派的过程

再看看运行阶段中虚拟机的选择，也就是**动态分派**的过程。

在执行 “son.hardChoice(new QQ()) ” 这行方法时，更准确的说，是在执行这行代码所对应的  Invokevirtual指令时，由于编译期已经决定目标方法的签名 必须为 hardChoice（QQ） ,虚拟机此时 不会关心传递过来的参数 “QQ”到底是 “腾讯QQ”还是“奇瑞QQ”，因为这时候参数的静态类型、实际类型 都对方法的选择不会构成任何影响，唯一可以影响虚拟机选择的因素只有该方法的接收者的实际类型是 Father 还是 Son 。

因为只有一个宗量作为选择依据，所以Java语言的动态分派属于单分派类型 。



根据上述论证的结果，我们可以总结一句： 

如今的Java语言是一门静态多分派、动态单分派的语言。

强调"如今的Java语言"是因为这个结论未必会恒久不变。JDK10 时Java语法中新出现 var关键字，

var 是在编译时根据声明语句中赋值符右侧的表达式类型来静态地推断类型。这本质时一种语法糖：



#### 虚拟机动态分派的实现

前面介绍的分派过程，作为对Java虚拟机概念模型的解释基本上已经足够了，它已经解决了虚拟机在分派中“**会做什么**”这问题，但是 问Java虚拟机 “**具体如果做到**” 的，答案则可能因为 各种虚拟机的实现不同会有些差别。

动态分派 是执行非常频繁的动作，而且动态分派的方法版本 选择过程 需要运行时在接收者类型的方法元数据中 选择合适的目标方法，因此Java虚拟机实现基于执行性能的考虑，真实运行时一般不会如此频繁的去反复搜索类型元数据。

面对这种情况，一种基础而且常见的优化手段 是为类型在方法中建一个虚方法表（Virtual Method Table，也称为 vtable），与此对应的，在invokeinterface 执行时也会用到 接口方法表-Interface Method Table，简称itable。 使用虚方法表 索引来代替元数据查找以提高性能。  我们来看看上面代码 所对应的虚方法表结构示例： 如下图所示

![](/uploads/jvm/11diapatch/methodTable.png)

虚方法表中存放着各个方法的实际入口地址，如果某个方法在子类中没有被重写，那子类的虚方法表中的地址入口和父类相同方法的地址入口是 一致的，都指向父类的实现入口。

如果子类中重写了这个方法，子类虚方法表中的地址也会被替换为 指向子类实现版本的入口地址。

在上图中，son重写了来自 Father的全部方法，因此son的方法表 没有指向Father类型数据的箭头。

但是son 和 Father都没有重写来自Object的方法，所以他们的方法表中所有从Object 继承来的方法都指向了Object的类型数据。





为了程序实现方便，具有相同签名的方法，在父类、子类的虚方法表中 都应当具有一样的索引序号，这样当类型变换时，仅仅需要变更查找的虚方法表，就可以从不同的虚方法表中按索引转换出所需的入口地址。 **虚方法表一般在类加载的连接阶段进行初始化**，准备了类的变量初始值后，虚拟机 会把 该**类的虚方法表也一同初始化完毕。**



上面说的**查虚方法表** 是 **分派调用**的一种**优化手段**，**由于Java对象里面的方法 默认（即不使用final 修饰）就是虚方法**，虚拟机除了使用虚方法表外，为了进一步提高性能，还会使用**类型继承分析（Class Hierarchy Analysis，CHA）** ，**守护内联（Guarded Inlining）**，**内联缓存（Inline Cache）** 等多种非稳定的激进优化来争取更大的性能空间，

