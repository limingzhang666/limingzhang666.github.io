---
title: 深入理解jvm-14-前端编译与优化
copyright: true
related_posts: true
date: 2021-01-27 20:12:07
tags: 前端编译与优化
categories: jvm

---

# 前端编译与优化

在Java技术下谈“编译期”而没有具体上下文语境的话，其实是一句很含糊的表述，

- 因为它可能是指一个前端编译器（叫“编译器的前端”更准确一些）把*.java文件转变成*.class文件的过程；

- 也可能是指Java虚拟机的即时编译器（常称JIT编译器，Just In Time Compiler）运行期把字节码转变成本地机器码的过程；

- 还可能是指使用静态的提前编译器（常称AOT编译器，Ahead Of Time Compiler）直接把程序编译成与目标机器指令集相关的二进制代码的过程

列举了这3类编译过程里一些比较有代表性的编译器产品：



- 前端编译器：JDK的Javac、Eclipse JDT中的增量式编译器（ECJ）
- 即时编译器：HotSpot虚拟机的C1、C2编译器，Graal编译器。
- 提前编译器：JDK的Jaotc、GNU Compiler for the Java（GCJ）、Excelsior JET

因为Java虚拟机设计团队选择把**对性能的优化全部集中到运行期的即时编译器中**，这样可以让那些不是由Javac产生的Class文件（如JRuby、Groovy等语言的Class
文件）也同样能享受到编译器优化措施所带来的性能红利。

- 前端编译器，开发优化程序员的编码过程，降低了编码复杂度，提高了编码效率，提高了程序员的幸福感，很多”**语法糖**（枚举、泛型、装箱拆箱）“就是依赖前端编译期实现的。
- 即时编译器 在运行期的优化过程，支撑了程序的执行效率不断的提升，



## Javac编译器

分析源码是了解一项技术的实现内幕最彻底的手段，Javac编译器不像HotSpot虚拟机那样使用C++语言（包含少量C语言）实现，它本身就是一个由Java语言编写的程序，这为纯Java的程序员了解它的编译过程带来了很大的便利。

#### Javac编译过程

1. 准备过程： 初始化插入式注解处理器
2. 解析与填充符号表过程，包括：
   - 词法、语法分析。将源代码的字符流转变为标记集合，构造出抽象语法树
   - 填充符号表。产生符号地址和符号信息
3. 插入式注解处理器的注解处理过程：  插入式注解处理器的执行阶段
4. 分析与字节码生成过程，包括：
   - 标注检查。对语法的静态信息进行检查。
   - 数据流及控制流分析。对程序动态运行过程进行检查
   - 解语法糖。将简化代码编写的语法糖还原为原有的形式
   - 字节码生成。将前面各个步骤所生成的信息转换为字节码

上述3个处理过程，执行插入式注解时又可能会产生新的符号，如果有新的符号产生，就必须转回到之前的解析、填充符号表的过程中 重新处理这些新符号。从总体上来看，三者之间的关系与交互顺序如下图所示

![](/uploads/jvm/13Compile/javac-compile.png)



- Javac编译动作的入口是com.sun.tools.javac.main.JavaCompiler类，上述3个过程的代码逻辑集中在这个类的compile()和compile2()方法里

![](/uploads/jvm/13Compile/javac-compile2.png)

#### 解析与填充符号表

解析过程由图10-5中的parseFiles()方法（图10-5中的过程1.1）来完成，解析过程包括了经典程序编译原理中的词法分析和语法分析两个步骤。

##### 词法、语法分析

**词法分析**是将源代码的字符流转变为标记（Token）集合的过程，单个字符是程序编写时的最小元素，但标记才是编译时的最小元素。关键字、变量名、字面量、运算符都可以作为标记，如“int a=b+2”这句代码中就包含了6个标记，分别是int、a、=、b、+、2，虽然关键字int由3个字符构成，但是它只是一个独立的标记，不可以再拆分。在Javac的源码中，词法分析过程由com.sun.tools.javac.parser.Scanner类来实现。



**语法分析**是根据标记序列构造抽象语法树的过程，抽象语法树（Abstract Syntax Tree，AST）是一种用来描述程序代码语法结构的树形表示方式，抽象语法树的每一个节点都代表着程序代码中的一个语法结构（Syntax Construct），例如包、类型、修饰符、运算符、接口、返回值甚至连代码注释等都可以是一种特定的语法结构。

在Javac的源码中，**语法分析过程由com.sun.tools.javac.parser.Parser**类实现，这个阶段产出的**抽象语法树是以com.sun.tools.javac.tree.JCTree类表示**的。

##### 填充符号表

完成了语法分析和词法分析之后，下一个阶段是对符号表进行填充的过程，也就是图10-5中 enterTrees()方法（图10-5中注释的过程1.2）要做的事情

符号表（Symbol Table）是由一组符号地址和符号信息构成的数据结构，读者可以把它类比想象成哈希表中键值对的存储形式（实际上符号表不一定是哈希表实现，可以是有序符号表、树状符号表、栈结构符号表等各种形式）。符号表中所登记的信息在编译的不同阶段都要被用到。譬如在语义分析的过程中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的声明是否一致）和产生中间代码，在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的直接依据。

在Javac源代码中，**填充符号表**的过程由**com.sun.tools.javac.comp.Enter类实现**，该过程的产出物是一个待处理列表，其中包含了每一个编译单元的抽象语法树的顶级节点，以及package-info.java（如果存在的话）的顶级节点。

##### 注解处理器（Lombok的实现原理）

在JDK 6中又提出并通过了JSR-269提案[1]，该提案设计了一组被称为“**插入式注解处理器**”的标准API，可以提前至编译期对代码中的特定注解进行处理，从而影响到前端编译器的工作过程。我们可以把插入式注解处理器看作是一组编译器的插件，当这些插件工作时，允许读取、修改、添加抽象语法树中的任意元素。如果这些插件在处理注解期间对语法树进行过修改，编译器将回到解析及填充符号表的过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止，每一次循环过程称为一个轮次（Round），这也就对应着图10-4的那个回环过程。

有了编译器注解处理的标准API后，**程序员的代码才有可能干涉编译器的行为**，**由于语法树中的任意元素，甚至包括代码注释都可以在插件中被访问到**，所以**通过插入式注解处理器实现的插件在功能上有很大的发挥空间**。只要有足够的创意，程序员能使用插入式注解处理器来实现许多原本只能在编码中由人工完成的事情。譬如Java著名的编码效率工具**Lombok**[2]，它可以通过注解来实现自动产生getter/setter方法、进行空置检查、生成受查异常表、产生equals()和hashCode()方法，等等，帮助开发人员消除Java的冗长代码，**这些都是依赖插入式注解处理器**来实现的，

在Javac源码中，插入式注解处理器的初始化过程是在**initPorcessAnnotations**()方法中完成的，而它的执行过程则是在**processAnnotations**()方法中完成。这个方法会判断是否还有新的注解处理器需要执行，如果有的话，通过com.sun.tools.javac.processing.JavacProcessing-Environment类的doProcessing()方法来生成一个新的JavaCompiler对象，对编译的后续步骤进行处理。

##### 语义分析与字节码生成（IDE工具提示报错原理）

经过语法分析之后，编译器获得了程序代码的抽象语法树表示，抽象语法树能够表示一个结构正确的源程序，但无法保证源程序的语义是符合逻辑的

而**语义分析**的主要任务则是对结构上正确的源程序**进行上下文相关性质的检查**，譬如进行**类型检查、控制流检查、数据流检查**。

我们编码时经常能在IDE中看到由红线标注的错误提示，其中绝大部分都是来源于语义分析阶段的检查结果。

###### 标注检查

Javac在编译过程中，语义分析过程可分为标注检查和数据及控制流分析两个步骤，分别由图10-5的attribute()和flow()方法（分别对应图10-5中的过程3.1和过程3.2）完成。

标注检查步骤要检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配，等等

在标注检查中，还会顺便进行一个称为常量折叠（Constant Folding）的代码优化，这是Javac编译器会对源代码做的极少量优化措施之一（代码优化几乎都在即时编译器中进行）

标注检查步骤在Javac源码中的实现类是com.sun.tools.javac.comp.Attr类和com.sun.tools.javac.comp.Check类。

###### 数据及控制流分析

数据流分析和控制流分析是对程序上下文逻辑更进一步的验证，它可以检查出诸如程序局部变量在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理了等问题。编译时期的数据及控制流分析与类加载时的数据及控制流分析的目的基本上可以看作是一致的，但校验范围会有所区别，有一些校验项只有在编译期或运行期才能进行。



下面举一个关于final修饰符的数据及控制流分析的例子，见代码清单10-1所示。

```java
// 方法一带有final修饰
public void foo(final int arg) {
    final int var = 0;
    // do something
}
// 方法二没有final修饰
public void foo(int arg) {
    int var = 0;
    // do something
}
```

在这两个foo()方法中，一个方法的参数和局部变量定义使用了final修饰符，另外一个则没有，**在代码编写时程序肯定会受到final修饰符的影响**，**不能再改变arg和var变量的值**，

但是如果观察这**两段代码编译出来的字节码**，会发现**它们是没有任何一点区别的**，每条指令，甚至每个字节都一模一样。

通过第6章对Class文件结构的讲解我们已经知道，局部变量与类的字段（实例变量、类变量）的存储是有显著差别的，**局部变量在常量池中并没有CONSTANT_Fieldref_info的符号引用**，**自然就不可能存储有访问标志（access_flags）的信息**，甚至可能连变量名称都不一定会被保留下来（这取决于编译时的
编译器的参数选项），自然**在Class文件中就不可能知道一个局部变量是不是被声明为final了**。

因此，可以肯定地推断出把局部变量声明为final，对运行期是完全没有影响的，**变量的不变性仅仅由Javac编译器在编译期间来保障**，这就是一个只能在编译期而不能在运行期中检查的例子。

在Javac的源码中，数据及控制流分析的入口是图10-5中的flow()方法（图10-5中的过程3.2），具体操作由com.sun.tools.javac.comp.Flow类来完成。

###### 解语法糖

语法糖（Syntactic Sugar），也称糖衣语法，，是由英国计算机科学家Peter J.Landin发明的一种编程术语，指的是在计算机语言中添加的某种语法，这种语法对语言的编译结果和功能并没有实际影响，但是却能更方便程序员使用该语言。通常来说使用语法糖能够减少代码量、增加程序的可读性，从而减少程序代码出错的机会。

Java中最常见的语法糖包括了前面提到过的泛型（其他语言中泛型并不一定都是语法糖实现，如C#的泛型就是直接由CLR支持的）、变长参数、自动装箱拆箱，等等，Java虚拟机运行时并不直接支持这些语法，它们在编译阶段被还原回原始的基础语法结构，这个过程就称为解语法糖.

在Javac的源码中，解语法糖的过程由desugar()方法触发，在com.sun.tools.javac.comp.TransTypes类
和com.sun.tools.javac.comp.Lower类中完成。

###### 字节码生成

字节码生成是Javac编译过程的最后一个阶段，在Javac源码里面由com.sun.tools.javac.jvm.Gen类来完成。字节码生成阶段不仅仅是把前面各个步骤所生成的信息（语法树、符号表）转化成字节码指令写到磁盘中，编译器还进行了少量的代码添加和转换工作.

- 前文多次登场的实例构造器<init>()方法和类构造器<clinit>()方法就是在这个阶段被添加到语法树之中的
- 请注意**这里的实例构造器并不等同于默认构造函数**，如果用户代码中没有提供任何构造函数，那编译器将会添加一个没有参数的、可访问性（public、protected、private或<package>）与当前类型一致的默认构造函数，**这个工作在填充符号表阶段中就已经完成**
- <init>()和<clinit>()这两个构造器的产生实际上是一种代码收敛的过程，编译器会把语句块（对于实例构造器而言是“{}”块，对于类构造器而言是“static{}”块）、变量初始化（实例变量和类变量）、调用父类的实例构造器（仅仅是实例构造器，<clinit>()方法中无须调用父类的<clinit>()方法，Java虚拟机会自动保证父类构造器的正确执行，但在<clinit>()方法中经常会生成调用java.lang.Object的<init>()方法的代码）等操作收敛到<init>()和
  <clinit>()方法之中，并且**保证无论源码中出现的顺序如何，都一定是按先执行父类的实例构造器，然后初始化变量，最后执行语句块的顺序进行**，上面所述的动作由**Gen::normalizeDefs()**方法来实现
- 还有其他的一些代码替换工作用于优化程序某些逻辑的实现方式，如把字符串的加操作替换为StringBuffer或StringBuilder（取决于目标代码的版本是否大于或等于JDK 5）的append()操作，等等。

完成了对语法树的遍历和调整之后，就会把填充了所有所需信息的符号表交到**com.sun.tools.javac.jvm.ClassWriter**类手上，由这个类的writeClass()方法输出字节码，**生成最终的Class文件**，到此，整个编译过程宣告结束。



### Java语法糖的味道

语法糖可以看作是前端编译器实现的一些“小把戏”，这些“小把戏”可能会使效率得到“大提升”，但我们也应该去了解这些“小把戏”背后的真实面貌，那样才能利用好它们，而不是被它们所迷惑。

**ps:  其实只需要在 反编译class文件后，就可以知道 使用了语法糖的java文件最后去语法糖后，变成了什么样子**

#### 泛型

泛型的本质是 参数化类型（Parameterized Type）或者参数化多态（Parameteric Polymorphism）的应用， 即可以**将操作的数据类型指定为方法签名中的一种参数**，这种参数类型能够用在 类、接口和方法的创建中，分别构成 泛型类、泛型接口和泛型方法。 泛型让程序员能够针对泛化的数据类型编写相同的算法，这极大的增强了编程语言的类型系统以及抽象能力。

Java的泛型直到今天依然作为Java语言不如C#语言好用的“铁证”被众人嘲讽

##### Java与C#的泛型

- Java选择的泛型实现方式叫作“类型擦除式泛型”（Type Erasure Generics），

- 而C#选择的泛型实现方式是“具现化式泛型”（Reified Generics）。具现化和特化、偏特化这些名词最初都是源于C++模版语法中的概念

- C#里面泛型无论在程序源码里面、编译后的中间语言表示（Intermediate Language，这时候泛型是一个占位符）里面，抑或是运行期的CLR里面都是切实存在的，List<int>与 List<string>就是两个不同的类型，它们由系统在运行期生成，有着自己独立的虚方法表和类型数据。
- 而Java语言中的泛型则不同，它只在程序源码中存在，在编译后的字节码文件中，全部泛型都被替换为原来的裸类型（Raw Type),并且在相应的地方插入了强制转型代码，因此对于运行期的Java语言来说，ArrayList<int>与ArrayList<String>其实是同一个类型，由此读者可以想象“类型擦除”这个名字的含义和来源

代码清单10-2　Java中不支持的泛型用法

```java
public class TypeErasureGenerics<E> {
    public void doSomething(Object item) {
        if (item instanceof E) { // 不合法，无法对泛型进行实例判断
        	...
        }
        E newItem = new E(); // 不合法，无法使用泛型创建对象
        E[] itemArray = new E[10]; // 不合法，无法使用泛型创建数组
    }
}
```

- C#2.0引入了泛型之后，带来的显著优势之一便是对比起Java在执行性能上的提高，因为在**使用平台提供的容器类型**（如List<T>，Dictionary<TKey，TValue>）时，**无须像Java里那样不厌其烦地拆箱和装箱**[1]，如果在Java中要避免这种损失，就必须构造一个与数据类型相关的容器类（譬如IntFloatHashMap这样的容器）。显然，这除了引入更多代码造成复杂度提高、复用性降低之外，更是丧失了泛型本身的存在价值。

- Java的类型擦除式泛型无论在使用效果上还是运行效率上，几乎是全面落后于C#的具现化式泛型，而它的**唯一优势是在于实现这种泛型的影响范围上**：擦除式泛型的**实现几乎只需要在Javac编译器上做出改进即可，不需要改动字节码、不需要改动Java虚拟机**，也**保证了以前没有使用泛型的库可以直接运行在Java 5.0之上**。但这种听起来节省工作量甚至可以说是有偷工减料嫌疑的优势就显得非常短视

在没有泛型的时代，由于Java中的数组是支持协变（Covariant）的[6]，对应的集合类也可以存入不同类型的元素，类似于代码清单10-3这样的代码尽管不提倡，但是完全可以正常编译成Class文件。

代码清单10-3　以下代码可正常编译为Class

```java
Object[] array = new String[10];
array[0] = 10; // 编译期不会有问题，运行时会报错
ArrayList things = new ArrayList();
things.add(Integer.valueOf(10)); //编译、运行时都不会报错
things.add("hello world");
```

为了保证这些编译出来的Class文件可以在Java 5.0引入泛型之后继续运行，设计者面前大体上有两条路可以选择：
1）需要泛型化的类型（主要是容器类型），以前有的就保持不变，然后平行地加一套泛型化版本的新类型。
2）直接把已有的类型泛型化，即让所有需要泛型化的已有类型都原地泛型化，不添加任何平行于已有类型的泛型版。

C# 选择了第一种方式，平行的加了一套泛型化版本的新类型 

Java选择了，第一要考虑到 保证“二进制向后兼容性”（Binary Backwards Compatibility）。二进制向后兼容性是明确写入《Java语言
规范》中的对Java使用者的严肃承诺，譬如一个在JDK 1.2中编译出来的Class文件，必须保证能够在JDK 12乃至以后的版本中也能够正常运行。

然后Java当时已经流行了10来年，存量老代码太多。

但**第二条路也并不意味着一定只能使用类型擦除来实现**，如果当时有足够的时间好好设计和实现，是完全有可能做出更好的泛型系统的，否则也不会有今天的**Valhalla**项目来**还以前泛型偷懒留下的技术债**了



##### 类型擦除

由于Java选择了第二条路，直接把已有的类型泛型化。要让所有需要泛型化的已有类型，譬如 ArrayList，原地泛型化后变成 ArrayList<T>,而且保证以前直接使用ArrayList的代码在泛型新版本里必须还能继续用这同一容器，这就必须让所有泛型化的实例，譬如 ArrayList<Integer>、ArrayList<String>这些全部自动成为
ArrayList的子类型才能可以，否则类型转换就是不安全的。由此 就引出了 **“裸类型”（Raw Type）**的概念，裸类型应被视为所有该类型泛型化实例的共同父类型（Super Type），

代码清单10-4　裸类型赋值

```java
ArrayList<Integer> ilist = new ArrayList<Integer>();
ArrayList<String> slist = new ArrayList<String>();
ArrayList list; // 裸类型
list = ilist;
list = slist;
```

接下来的问题是该如何实现裸类型。这里又有了两种选择：

- 一种是在运行期由Java虚拟机来自动地、真实地构造出ArrayList<Integer>这样的类型，并且自动实现从ArrayList<Integer>派生自ArrayList的继承关系来满足裸类型的定义；
- （**被选择**）另外一种是**索性简单粗暴地直接在编译时把ArrayList<Integer>还原回ArrayList**，只在**元素访问、修改时自动插入一些强制类型转换和检查指令**，这样看起来也是能满足需要，这两个选择的最终结果大家已经都知道了。

代码清单10-5是一段简单的Java泛型例子，我们可以看一下它编译后的实际样子是怎样的。

代码清单10-5　泛型擦除前的例子

代码清单10-5　泛型擦除前的例子

```java
public static void main(String[] args) {
    Map<String, String> map = new HashMap<String, String>();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println(map.get("hello"));
    System.out.println(map.get("how are you?"));
}
```

把这段Java代码编译成Class文件，然后再用字节码反编译工具进行反编译后，将会发现泛型都不见了，程序又变回了Java泛型出现之前的写法，**泛型类型都变回了裸类型**，只在**元素访问时插入了从Object到String的强制转型代码**，如代码清单10-6所示。



代码清单10-6　泛型擦除后的例子

```java
public static void main(String[] args) {
    Map map = new HashMap();
    map.put("hello", "你好");
    map.put("how are you?", "吃了没？");
    System.out.println((String) map.get("hello"));
    System.out.println((String) map.get("how are you?"));
}
```

- 首先，使用擦除法实现泛型直接导致了对原始类型（Primitive Types）数据的支持又成了新的麻烦，譬如将代码清单10-2稍微修改一下，变成代码清单10-7这个样子

代码清单10-7　原始类型的泛型（目前的Java不支持）

```java
ArrayList<int> ilist = new ArrayList<int>();
ArrayList<long> llist = new ArrayList<long>();
ArrayList list;
list = ilist;
list = llist;
```

这种情况下，一旦把泛型信息擦除后，到要插入强制转型的地方就没办法往下做了，因为 不支持 int、long 与 Object之间的强制类型转换。 当时Java给出的解决方案一如既往的简单粗暴：  既然没法转换，**索性就不支持原生类型的泛型**。都用ArrayList<Integer>、ArrayList<Long>，反正都做了**自动的强制类型转换**，遇到**原生类型时把装箱、拆箱也自动做了**.。这个决定也导致了无数构造包装类的装箱、拆箱的开销， 称为Java泛型慢的重要原因。

- 第二，运行期无法取到泛型类型信息，会让一些代码变得相当啰嗦，譬如代码清单10-2中罗列的几种Java不支持的泛型用法，都是由于运行期Java虚拟机无法取得泛型类型而导致的。像代码清单10-8这样，我们去写一个泛型版本的从List到数组的转换方法，由于不能从List中取得参数化类型T，所以不得不从一个额外参数中再传入一个数组的组件类型进去，实属无奈。

代码清单10-8　不得不加入的类型参数

```java
public static <T> T[] convert(List<T> list, Class<T> componentType) {
    T[] array = (T[])Array.newInstance(componentType, list.size());
    ...
}
```

- 最后，通过擦除法来实现泛型，还丧失了一些面向对象思想应有的优雅，带来了一些模棱两可的模糊状况，例如代码清单10-9的例子

代码清单10-9　当泛型遇见重载1 (**这段代码无法编译通过**)

```java
public class GenericTypes {
    public static void method(List<String> list) {
   	 	System.out.println("invoke method(List<String> list)");
    }
    public static void method(List<Integer> list) {
    	System.out.println("invoke method(List<Integer> list)");
	}
}
```

这段代码是不能被编译的，因为参数List<Integer>和List<String>编译之后都被擦除了，变成了同一种的裸类型List，
类型擦除导致这两个方法的特征签名变得一模一样。（**特征签名最重要的任务就是作为方法独一无二不可重复的ID，在Java代码中的方法特征签名只包括了方法名称、参数顺序及参数类型，而在字节码中的特征签名还包括方法返回值及受查异常表**）

- 由于Java泛型的引入，各种场景（虚拟机解析、反射等）下的方法调用都可能对原有的基础产生影响并带来新的需求，如在泛型类中如何获取传入的参数化类型等。
- 所以JCP组织对《Java虚拟机规范》做出了相应的修改，引入了诸如Signature、LocalVariableTypeTable等新的属性用于解决伴随泛型而来的参数类型的识别问题，
- Signature是其中最重要的一项属性，它的作用就是存储一个方法在字节码层面的特征签名[8]，这个属性中保存的参数类型并不是原生类型，而是包括了参数化类型的信息。修改后的虚拟机规范[9]要求所有能识别49.0以上版本的Class文件的虚拟机都要能正确地识别Signature参数。
- 另外，从Signature属性的出现我们还可以得出结论，擦除法所谓的擦除，仅仅是对方法的Code属性中的字节码进行擦除，实际上元数据中还是保留了泛型信息，这也是我们在编码时能通过反射手段取得参数化类型的根本依据。

##### 值类型与未来的泛型

值类型，这点也是C#用户攻讦Java语言的常用武器之一，C#并没有Java意义上的原生数据类型，在C#中使用的int、bool、double关键字其实是对应了一系列在.NET框架中预定义好的结构体（Struct），如Int32、Boolean、Double等。在C#中开发人员也可以定义自己值类型，只要继承于ValueType类型即可，而ValueType也是统一基类Object的子类，所以并不会遇到Java那样int不自动装箱就无法转型为Object的尴尬。

值类型可以与引用类型一样，具有构造函数、方法或是属性字段，等等，而它与引用类型的区别在于它在赋值的时候通常是整体复制，而不是像引用类型那样传递引用的。更为关键的是，值类型的实例很容易实现分配在方法的调用栈上的，这意味着值类型会随着当前方法的退出而自动释放，不会给垃圾收集子系统带来任何压力。

##### 自动装箱、拆箱与遍历循环

代码清单10-11　自动装箱、拆箱与遍历循环

```java
public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4);
    int sum = 0;
    for (int i : list) {
    	sum += i;
    }
    System.out.println(sum);
}
```

代码清单10-12　自动装箱、拆箱与遍历循环编译之后

```java
public static void main(String[] args) {
    List list = Arrays.asList( new Integer[] {
    Integer.valueOf(1),
    Integer.valueOf(2),
    Integer.valueOf(3),
    Integer.valueOf(4) });
    int sum = 0;
    for (Iterator localIterator = list.iterator(); localIterator.hasNext(); ) {
        int i = ((Integer)localIterator.next()).intValue();
        sum += i;
    }
    System.out.println(sum);
}
```

代码清单10-11中一共包含了泛型、自动装箱、自动拆箱、遍历循环与变长参数5种语法糖，代码
清单10-12则展示了它们在编译前后发生的变化。泛型就不必说了，

- 自动装箱、拆箱在编译之后被转化成了对应的包装和还原方法，如本例中的Integer.valueOf()与Integer.intValue()方法，
- 而遍历循环则是把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口的原因。
- 最后再看看变长参数，它在调用的时候变成了一个数组类型的参数，在变长参数出现之前，程序员的确也就是使用数组来完成类似功能的。

这些语法糖虽然看起来很简单，但也不见得就没有任何值得我们特别关注的地方，代码清单10-13演示了自动装箱的一些错误用法



```java
public class AutoBoxing {
    public static void main(String[] args) {
        Integer a = 1;
        Integer b = 2;
        Integer c = 3;
        Integer d = 3;
        Integer e = 321;
        Integer f = 321;
        Long g = 3L;
        System.out.println(c == d);
        System.out.println(c+" "+d+" "+System.identityHashCode(c)+" "+System.identityHashCode(d));
        System.out.println(e == f);
        System.out.println(e+" "+f+" "+System.identityHashCode(e)+" "+System.identityHashCode(f));
        System.out.println(c == (a + b));
        System.out.println(c.equals(a + b));
        System.out.println(g == (a + b));
        System.out.println(g.equals(a + b));
    }
}


	// Integer 源码（解释为什么  System.out.println(e == f);  //返回false）
 	static final int low = -128;
        static final int high;
	//
    @HotSpotIntrinsicCandidate
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

鉴于**包装类的“==”运算在不遇到算术运算的情况下不会自动拆箱**，以及**它们equals()方法不处理数据转型的关系**，


Integer::valueOf(int i),  当i的值 处于[-128,127]时，从缓存Cache中取， 否则 new Integer(i)，包装类==比较hash地址，所以  返回false

##### 条件编译

在Java语言之中并没有使用预处理器，因为Java语言天然的编译方式（编译器并非一个个地编译Java文件，而是将所有编译单元的语法树顶级节点输入到待处理列表后再进行编译，因此各个文件之间能够互相提供符号信息）就无须使用到预处理器。

Java语言当然也可以进行条件编译，方法就是使用条件为常量的if语句。如代码清单10-14所示，该代码中的if语句不同于其他Java代码，它在编译阶段就会被“运行”，生成的字节码之中只包括“System.out.println("block 1")；”一条语句，并不会包含if语句及另外一个分子中的“System.out.println("block 2")；



```java
public static void main(String[] args) {
    if (true) {
    	System.out.println("block 1");
    } else {
    	System.out.println("block 2");
    }
}

// 编译后的反编译结果：
 public static void main(String[] args) {
	System.out.println("block 1");
}
    
```

只能使用条件为常量的if语句才能达到上述效果，如果使用常量与其他带有条件判断能力的语句搭配，则可能在控制流分析中提示错误，被拒绝编译，如代码清单10-15所示的代码就会被编译器拒绝编译。

代码清单10-15　不能使用其他条件语句来完成条件编译

```java
public static void main(String[] args) {
    // 编译器将会提示“Unreachable code”
    while (false) {
   	 System.out.println("");
    }
}
```

除了本节中介绍的泛型、自动装箱、自动拆箱、遍历循环、变长参数和条件编译之外，Java语言还有不少其他的语法糖，如**内部类、枚举类、断言语句、数值字面量、对枚举和字符串的switch支持、try语句中定义和关闭资源（这3个从JDK 7开始支持）、Lambda表达式（**从JDK 8开始支持，Lambda不能算是单纯的语法糖，但在前端编译器中做了大量的转换工作），等等，读者**可以通过跟踪Javac源码、反编译Class文件等方式了解它们的本质实现**

### 实战：插入式注解处理器(TODO)

NameCheckProcessor的实战例子只演示了JSR-269嵌入式注解处理API其中的一部分功能，基于这组API支持的比较有名的项目还有用于校验Hibernate标签使用正确性的Hibernate Validator AnnotationProcessor[1]（本质上与NameCheckProcessor所做的事情差不多）、自动为字段生成getter和setter方法等辅助内容的Lombok[2]（根据已有元素生成新的语法树元素）等，读者有兴趣的话可以参考它们官方站点的相关内容

## 总结：

在本章中，我们从Javac编译器源码实现的层次上学习了Java源代码编译为字节码的过程，分析了Java语言中泛型、主动装箱拆箱、条件编译等多种语法糖的前因后果，并实战练习了如何使用插入式注解处理器来完成一个检查程序命名规范的编译器插件。

如本章概述中所说的，在**前端编译器中，“优化”手段主要用于提升程序的编码效率**，之所以把Javac这类将Java代码转变为字节码的编译器称作“前端编译器”，是因为**它只完成了从程序到抽象语法树或中间字节码的生成**，而在此之后，还有一组**内置于Java虚拟机内部的“后端编译器”来完成代码优化以及从字节码生成本地机器码的过程**，即前面多次提到的即时编译器或提前编译器，这个**后端编译器的编译速度及编译结果质量高低，是衡量Java虚拟机性能最重要的一个指标**

