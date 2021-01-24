---
title: 深入理解jvm-12-动态类型语言支持
copyright: true
related_posts: true
date: 2021-01-24 16:00:57
tags: 动态类型语言支持
categories: jvm
---



# 动态类型语言支持

在介绍Java虚拟机的动态类型语言支持之前，

- 我们要先弄明白动态类型语言是什么？
- 它与Java语言、Java虚拟机有什么关系？

了解Java虚拟机提供动态类型语言支持的技术背景，对理解这个语言特性是非常有必要的。



## 动态类型语言

- 定义： 动态类型语言的关键特征是 它的**类型检查的主体是在运行期而不是 编译期进行的**，

- 满足这个特征的语言有很多，常用的包括： APL、Clojure、Erlang、Groovy、JavaScript、Lisp、Lua、PHP、Prolog、Python、Ruby、Smalltalk、Tcl，等等
- 那相对地，**在编译期就进行类型检查过程的语言**，譬如 C++ 和Java等就是最常用的**静态类型语言** 

```bash
public static void main(String[] args) {
	int[][][] array = new int[1][0][-1];
}
```

- 上面这段Java代码能够正常编译，但运行的时候会出现NegativeArraySizeException异常。在《Java虚拟机规范》中明确规定了NegativeArraySizeException是一个运行时异常（Runtime Exception），通俗一点说，**运行时异常就是指只要代码不执行到这一行就不会出现问题**。
- 与运行时异常相对应的概念是**连接时**（**类加载阶段的验证、准备和解析 称为连接阶**段）**异常**，例如很常见的NoClassDefFoundError便属于连接时异常，即使导致连接时异常的代码放在一条根本无法被执行到的路径分支上，类加载时（第7章解释过Java的连接过程不在编译阶段，而在类加载阶段）也照样会抛出异常。

不过在C语言中，语义相同的代码就会在编译期就直接出错，而不是等到运行时才出现异常：

```bash
int main(void) {
	int i[1][0][-1]; // GCC拒绝编译，报“size of array is negative”
	return 0;
}
```

由此看来，一门语言的哪一种检查行为 要在运行期进行，哪一种检查要在编译期进行并没有什么必然的因果逻辑关系，关键在于语言规范中人为设立的约定。

### 类型检查

```bash
obj.println("hello world");
```



虽然正在阅读本书的每一位读者都能看懂这行代码要做什么，但对于计算机来讲，这一行“没头没尾”的代码是无法执行的，它需要一个具体的上下文中（譬如程序语言是什么、obj是什么类型）才有讨论的意义。

- 现在先假设这行代码是在**Java语言中**，并且变量obj的静态类型为java.io.PrintStream，那变量obj的实际类型就必须是PrintStream的子类（实现了PrintStream接口的类）才是合法的。否则，哪怕obj属于一个确实包含有println(String)方法相同签名方法的类型，但只要它与PrintStream接口没有继承关系，代码依然不可能运行——因为***类型检查不合法***
- 但是相同的代码在**ECMAScript（JavaScript**）中情况则不一样，无论obj具体是何种类型，无论其继承关系如何，只要这种类型的方法定义中确实包含有println(String)方法，能够找到相同签名的方法，调用便可成功。
- 产生这种差别产生的根本原因是Java语言在编译期间却已将println(String)方法完整的符号引用（本例中为一项CONSTANT_InterfaceMethodref_info常量）生成出来，并作为方法调用指令的参数存储到Class文件中，
- 例如下面这个样子：

```bash
invokevirtual #4; //Method java/io/PrintStream.println:(Ljava/lang/String;)V
```

1. 这个符号引用包含了该方法定义在哪个具体类型之中、方法的名字以及参数顺序、参数类型和方法返回值等信息，通过这个符号引用，Java虚拟机就可以翻译出该方法的直接引用。
2. 而ECMAScript等动态类型语言与Java有一个核心的差异就是变量obj本身并没有类型，**变量obj的值才具有类型**，
3. 所以编译器在编译时最多只能确定方法名称、参数、返回值这些信息，而不会去确定方法所在的具体类型（即**方法接收者不固定**）。
4. ***“变量无类型而变量值才有类型”这个特点也是动态类型语言的一个核心特征。***



### 动态类型对比静态类型语言的优缺点：

了解了动态类型和静态类型语言的区别后，也许读者的下一个问题就是动态、静态类型语言两者谁更好，或者谁更加先进呢？这种比较不会有确切答案，它们都有自己的优点，选择哪种语言是需要权衡的事情。

- 静态类型语言能够**在编译期确定变量类型**，最显著的好处是**编译器可以提供全面严谨的类型检查**，这样与数据类型相关的潜在问题就能在编码时被及时发现，利于**稳定性**及让项目容易达到更大的规模。
- 而动态类型语言在运行期才确定类型，这可以为开发人员提供极大的**灵活性**，某些在静态类型语言中要花大量臃肿代码来实现的功能，由动态类型语言去做可能会很**清晰简洁**，清晰简洁通常也就意味着**开发效率的提升**。

- ps： **动态类型语言与动态语言、弱类型语言并不是一个概念，需要区别对待。**





## Java与动态类型

### 问题：

1. Java语言、Java虚拟机与 动态类型语言之间有什么关系。

- Java虚拟机毫无疑问是Java语言的运行平台，但它的使命并不限于此。

- 目前确实已经有许多动态类型语言运行于Java虚拟机之上了，如Clojure、Groovy、Jython和JRuby等，能够在同一个虚拟机之上可以实现静态类型语言的严谨与动态类型语言的灵活，这的确是一件很美妙的事情。
- 但遗憾的是Java虚拟机层面对动态类型语言的支持一直都还有所欠缺，主要表现在方法调用方面：
  - JDK 7以前的字节码指令集中，4条方法调用指令（invokevirtual、invokespecial、invokestatic、invokeinterface）的第一个参数都是**被调用的方法的符号引用**（CONSTANT_Methodref_info或者CONSTANT_InterfaceMethodref_info常量）
  - 前面已经提到过，方法的符号引用在编译时产生，而**动态类型语言只有在运行期才能确定方法的接收者**。这样，在Java虚拟机上实现的动态类型语言就不得不使用“曲线救国”的方式（如编译时留个占位符类型，运行时动态生成字节码实现具体类型到占位符类型的适配）来实现
  - 但这样势必会**让动态类型语言实现的复杂度增加**，也会带来额外的性能和内存开销。
  - 内存开销是很显而易见的，方法调用产生的那一大堆的动态类就摆在那里。
  - 而其中最严重的性能瓶颈是在于动态类型方法调用时，由于**无法确定调用对象的静态类型**，而导致的**方法内联无法有效进行**。
  - 尽管也可以想一些办法（譬如调用点缓存）尽量缓解支持动态语言而导致的性能下降，但这种改善毕竟不是本质的

譬如有类似一下代码：

```bash
var arrays = {"abc", new ObjectX(), 123, Dog, Cat, Car..}
for(item in arrays){
	item.sayHello();
}
```

### JVM动态字节码指令背景

- 在动态类型语言下这样的代码是没有问题，但由于在运行时arrays中的元素可以是任意类型，即使它们的类型中都有sayHello()方法，也肯定无法在编译优化的时候就确定具体sayHello()的代码在哪里，编译器只能不停编译它所遇见的每一个sayHello()方法，并缓存起来供执行时选择、调用和**内联，**
- 如果arrays数组中不同类型的对象很多，就势必会对内联缓存产生很大的压力，缓存的大小总是有限的，类型信息的不确定性导致了缓存内容不断被失效和更新，先前优化过的方法也可能被不断替换而无法重复使用。
- 所以这种**动态类型方法调用的底层问题**终归是**应当在Java虚拟机层次上去解决才最合适**。
- 因此，在Java虚拟机层面上提供动态类型的直接支持就成为Java平台发展必须解决的问题，
- 这便是JDK 7时JSR-292提案中**invokedynamic指令**以及 **java.lang.invoke** 包出现的技术背景。



## java.lang.invoke包

### 方法句柄：

JDK 7时新加入的java.lang.invoke包[1]是JSR 292的一个重要组成部分，这个包的主要目的是在之前单纯依靠符号引用来确定调用的目标方法这条路之外，提供一种新的动态确定目标方法的机制，称为“方法句柄”（Method Handle）。

不妨把方法句柄与C/C++ 中的函数指针（Function Pointer），或者C# 里面的委派（Delegate）互相类比一下来理解。举个例子，，如果我们实现一个带谓词（谓词就是外部传入的排序时比较大小的动作）的排序函数，在C/C++ 中的常用做法时把谓词定义为函数，用函数指针来把谓词传递到排序方法，像这样：



```bash
void sort(int list[], const int size, int (*compare)(int, int))
```

但在Java语言中做不到这一点，没有办法单独把一个函数作为参数进行传递，普遍的做法是  涉及一个带有compare（）方法的Comparetor 接口，以实现这个接口的对象作为参数，例如 Java类库中的Collections：： sort 方法就是这样定义的：

```java
void sort(List list,Comparator c)
```

不过，在拥有方法句柄之后，Java 语言也可以拥有类似于 函数指针或者委托的方法别名这样的工具了，

以下代码演示了方法句柄的基本用法，无论obj 是何种类型（临时定义的ClassA 或者是实现PrintStream接口的实现类 System.out），都可以正确的调用到println（）方法



```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;

import static java.lang.invoke.MethodHandles.lookup;

/**
 * JSR 292 MethodHandle基础用法演示
 * @author zzm
 */
public class MethodHandleTest {

    static class ClassA {
        public void println(String s) {
            System.out.println(s);
        }
    }

    public static void main(String[] args) throws Throwable {
        Object obj = System.currentTimeMillis() % 2 == 0 ? System.out : new ClassA();
        // 无论obj最终是哪个实现类，下面这句都能正确调用到println方法。
        getPrintlnMH(obj).invokeExact("icyfenix");
    }

    private static MethodHandle getPrintlnMH(Object reveiver) throws Throwable {
        // MethodType：代表“方法类型”，包含了方法的返回值（methodType()的第一个参数）和具体参数（methodType()第二个及以后的参数）。
        MethodType mt = MethodType.methodType(void.class, String.class);
        // lookup()方法来自于MethodHandles.lookup，这句的作用是在指定类中查找符合给定的方法名称、方法类型，并且符合调用权限的方法句柄。
        // 因为这里调用的是一个虚方法，按照Java语言的规则，方法第一个参数是隐式的，代表该方法的接收者，也即是this指向的对象，这个参数以前是放在参数列表中进行传递，现在提供了bindTo()方法来完成这件事情。
        return lookup().findVirtual(reveiver.getClass(), "println", mt).bindTo(reveiver);
    }
}
```

方法getPrintlnMH()中实际上是模拟了invokevirtual指令的执行过程，只不过它的分派逻辑并非固化在Class文件的字节码上，而是通过一个由用户设计的Java方法来实现。

而这个方法本身的返回值（MethodHandle对象），可以视为对最终调用方法的一个“引用”。以此为基础，有了MethodHandle就可以写出类似于C/C++那样的函数声明了：

```java
void sort(List list, MethodHandle compare)
```

从上面的例子看来，使用MethodHandle并没有多少困难，不过看完它的用法之后，读者大概就会产生疑问，相同的事情，用反射不是早就可以实现了吗？

确实，仅站在Java语言的角度看，MethodHandle在使用方法和效果上与Reflection有众多相似之处。不过，它们也有以下这些区别：

1. Reflection和MethodHandle机制本质上都是在模拟方法调用，但是**Reflection是在模拟Java代码层次的方法调用**，而**MethodHandle是在模拟字节码层次的方法调用**。在MethodHandles.Lookup上的3个方法findStatic()、findVirtual()、findSpecial()正是为了对应于invokestatic、invokevirtual（以及
   invokeinterface）和invokespecial这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
2. Reflection中的java.lang.reflect.Method对象远比MethodHandle机制中的java.lang.invoke.MethodHandle对象所包含的信息来得多。前者是**方法在Java端的全面映像，包含了方法的签名、描述符以及方法属性表中各种属性的Java端表示方式，还包含执行权限等的运行期信息**。而后者仅包含**执行该方法的相关信息**。用开发人员通俗的话来讲，**Reflection是重量级，而MethodHandle是轻量级。**
3. 由于MethodHandle是对字节码的方法指令调用的模拟，那理论上虚拟机在这方面做的各种优化（如方法内联），在MethodHandle上也应当可以采用类似思路去支持（但目前实现还在继续完善中），而通过**反射去调用方法则几乎不可能直接去实施各类调用点优化措施**。
4. MethodHandle与Reflection除了上面列举的区别外，最关键的一点还在于去掉前面讨论施加的前提“仅站在Java语言的角度看”之后：**Reflection API的设计目标是只为Java语言服务的**，而**MethodHandle则设计为可服务于所有Java虚拟机之上的语言，其中也包括了Java语言而已**，而且Java在这里并不是主角。



## invokedynamic指令

了JDK 7为了更好地支持动态类型语言，引入了第五条方法调用的字节码指令invokedynamic，

- 某种意义上可以说invokedynamic指令与MethodHandle机制的作用是一样的，都是为了解决原有4条“invoke*”指令方法（***invokevirtual***、***invokespecial***、***invokestatic***、***invokeinterface***））分派规则完全固化在虚拟机之中的问题，把**如何查找目标方法的决定权从虚拟机转嫁到具体用户代码之中**，让用户（广义的用户，包含其他程序语言的设计者）有更高的自由度。
- 而且，它们两者的思路也是可类比的，都是为了达成同一个目的，只是一个用上层代码和API来实现，另一个用字节码和Class中其他属性、常量来完成。因此，如果前面MethodHandle的例子看懂了，相信理解invokedynamic指令并不困难。



- 每一处含有invokedynamic指令的位置都被称作“**动态调用点（Dynamically-Computed Call Site）**”，这条指令的第一个参数不再是代表方法符号引用的CONSTANT_Methodref_info常量，而是**变为JDK 7时新加入的CONSTANT_InvokeDynamic_info常量**，从这个新常量中可以得到3项信息：**引导方法**
  **（Bootstrap Method，该方法存放在新增的BootstrapMethods属性中）、方法类型（MethodType）和名称。**
- 引导方法是有固定的参数，并且返回值规定是java.lang.invoke.CallSite对象，这个对象代表了真正要执行的目标方法调用。根据CONSTANT_InvokeDynamic_info常量中提供的信息，虚拟机可以找到并且执行引导方法，从而获得一个CallSite对象，最终调用到要执行的目标方法上

代码清单8-13所示

```java
import java.lang.invoke.*;

import static java.lang.invoke.MethodHandles.lookup;

public class InvokeDynamicTest {

    public static void main(String[] args) throws Throwable {
        INDY_BootstrapMethod().invokeExact("icyfenix");
    }

    public static void testMethod(String s) {
        System.out.println("hello String:" + s);
    }

    public static CallSite BootstrapMethod(MethodHandles.Lookup lookup, String name, MethodType mt) throws Throwable {
        return new ConstantCallSite(lookup.findStatic(InvokeDynamicTest.class, name, mt));
    }

    private static MethodType MT_BootstrapMethod() {
        return MethodType
                .fromMethodDescriptorString(
                        "(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;",
                        null);
    }

    private static MethodHandle MH_BootstrapMethod() throws Throwable {
        return lookup().findStatic(InvokeDynamicTest.class, "BootstrapMethod", MT_BootstrapMethod());
    }

    private static MethodHandle INDY_BootstrapMethod() throws Throwable {
        CallSite cs = (CallSite) MH_BootstrapMethod().invokeWithArguments(lookup(), "testMethod",
                MethodType.fromMethodDescriptorString("(Ljava/lang/String;)V", null));
        return cs.dynamicInvoker();
    }
}
```

- 前文提到过，由于***invokedynamic***指令面向的主要服务对象并非Java语言，而是**其他Java虚拟机之上的其他动态类型语言**，因此，光靠Java语言的
  编译器Javac的话，在JDK 7时甚至还完全没有办法生成带有invokedynamic指令的字节码（曾经有一个java.dyn.InvokeDynamic的语法糖可以实现，但后来被取消了），
- 而到JDK 8引入了Lambda表达式和接口默认方法后，Java语言才算享受到了一点invokedynamic指令的好处，但用Lambda来解释invokedynamic指令运作就比较别扭，也无法与前面MethodHandle的例子对应类比，
- 所以笔者采用一些变通的办法：John Rose（JSR 292的负责人，以前Da Vinci Machine Project的Leader）编写过一个把程序的字节码转换为使用invokedynamic的简单工具**INDY**[1]来完成这件事，我们要使用这个工具来产生最终需要的字节码，
- 因此代码清单8-13中的方法名称不能随意改动，更不能把几个方法合并到一起写，因为它们是要被INDY工具读取的

- 把上面的代码编译，再使用INDY转换后重新生成的字节码如代码清单8-14所示,使用javap输出

代码清单8-14　InvokeDynamic指令演示（2）

```java
Constant pool:
    #121 = NameAndType #33:#30 // testMethod:(Ljava/lang/String;)V
    #123 = InvokeDynamic #0:#121 // #0:testMethod:(Ljava/lang/String;)V
public static void main(java.lang.String[]) throws java.lang.Throwable;
	Code:
		stack=2, locals=1, args_size=1
            0: ldc #23 // String abc
            2: invokedynamic #123, 0 // InvokeDynamic #0:testMethod: (Ljava/lang/String;)V   
            7: nop
            8: return
public static java.lang.invoke.CallSite BootstrapMethod(java.lang.invoke.Method Handles$Lookup, java.lang.String, 
	Code:
		stack=6, locals=3, args_size=3
            0: new #63 // class java/lang/invoke/ConstantCallSite
            3: dup
            4: aload_0
            5: ldc #1 // class org/fenixsoft/InvokeDynamicTest
            7: aload_1
            8: aload_2
            9: invokevirtual #65 // Method java/lang/invoke/MethodHandles$ Lookup.findStatic:(Ljava/lang/Class;
            12: invokespecial #71 // Method java/lang/invoke/ConstantCallSite. "<init>":(Ljava/lang/invoke/MethodHandle;)
            15: areturn
```

从main()方法的字节码中可见，原本的方法调用指令已经被替换为invokedynamic了，它的参数为第123项常量（第二个值为0的参数在虚拟机中不会直接用到，这与invokeinterface指令那个的值为0的参数一样是占位用的，目的都是为了给常量池缓存留出足够的空间）：

```java
2: invokedynamic #123, 0 // InvokeDynamic #0:testMethod:(Ljava/lang/String;)V
```

从常量池中可见，第123项常量显示“#123=InvokeDynamic#0：

- #121”说明它是一项CONSTANT_InvokeDynamic_info类型常量，常量值中前面“#0”代表引导方法取Bootstrap Methods属性表的第0项（javap没有列出属性表的具体内容，不过示例中仅有一个引导方法，即BootstrapMethod()），
- 而后面的“#121”代表引用第121项类型为CONSTANT_NameAndType_info的常量，从这个常量中可以获取到方法名称和描述符，即后面输出的“testMethod：
  (Ljava/lang/String；)V”。



再看BootstrapMethod()，这个方法在Java源码中并不存在，是由INDY产生的，但是它的字节码很容易读懂，

- 所有逻辑都是调用MethodHandles$Lookup的findStatic()方法，产生testMethod()方法的MethodHandle，
- 然后用它创建一个ConstantCallSite对象。
- 最后，这个对象返回给invokedynamic指令实现对testMethod()方法的调用，invokedynamic指令的调用过程到此就宣告完成了。



## 实战：掌控方法分派规则

**invokedynamic**指令与此前4条传统的“invoke*”指令的最大区别就是它的**分派逻辑不是由虚拟机决定的**，而是**由程序员决定**。

在介绍Java虚拟机动态语言支持的最后一节中，笔者希望通过一个简单例子（如代码清单8-15所示），帮助读者理解程序员可以掌控方法分派规则之后，我们能做什么以前无法做到的事情。

代码清单8-15　方法调用问题

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;

import static java.lang.invoke.MethodHandles.lookup;

/**
 * @author zzm
 */
public class GrandFatherTestCase_1 {

    class GrandFather {
        void thinking() {
            System.out.println("i am grandfather");
        }
    }

    class Father extends GrandFather {
        void thinking() {
            System.out.println("i am father");
        }
    }

    class Son extends Father {
        void thinking() {
            // 请读者在这里填入适当的代码（不能修改其他地方的代码）
			// 实现调用祖父类的thinking()方法，打印"i am grandfather"
        }
    }

    public static void main(String[] args) {
        (new GrandFatherTestCase_1().new Son()).thinking();
    }

}
```

在Java程序中，可以通过“super”关键字很方便地调用到父类中的方法，但如果要访问祖类的方法呢？不妨自己思考一下，在JDK 7之前有没有办法解决这
个问题。

在拥有invokedynamic和java.lang.invoke包之前，使用纯粹的Java语言很难处理这个问题（使用ASM等字节码工具直接生成字节码当然还是可以处理的，但这已经是在字节码而不是Java语言层面来解决问题了），原因是在Son类的thinking()方法中根本无法获取到一个实际类型是GrandFather的对象引用，而invokevirtual指令的分派逻辑是固定的，只能按照方法接收者的实际类型进行分派，这个逻辑完全固化在虚拟机中，程序员无法改变。

如果是JDK 7 Update 9之前，使用代码清单8-16中的程序就可以直接解决该问题。

代码清单8-16　使用MethodHandle来解决问题

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodType;

import static java.lang.invoke.MethodHandles.lookup;

/**
 * @author zzm
 */
public class GrandFatherTestCase_1 {

    class GrandFather {
        void thinking() {
            System.out.println("i am grandfather");
        }
    }

    class Father extends GrandFather {
        void thinking() {
            System.out.println("i am father");
        }
    }

    class Son extends Father {
        void thinking() {
            try {
                MethodType mt = MethodType.methodType(void.class);
                MethodHandle mh = lookup().findSpecial(GrandFather.class,
                        "thinking", mt, getClass());
                mh.invoke(this);
            } catch (Throwable e) {
            }
        }
    }

    public static void main(String[] args) {
        (new GrandFatherTestCase_1().new Son()).thinking();
    }

}


```

使用**JDK 7 Update 9之前**的HotSpot虚拟机运行，会得到如下运行结果： i am grandfather

ps:  **我试了一下用jdk 9**，运行结果是： i am father，因为: 

- 这个逻辑在JDK 7 Update 9之后被视作一个潜在的安全性缺陷修正了，
- 原因是**必须保证findSpecial()查找方法版本时受到的访问约束**（譬如对访问控制的限制、对参数类型的限制）**应与使用invokespecial指令一样**，两者必须保持精确对等，包括在上面的场景中它只能访问到其直接父类中的方法版本。
- 所以在JDK 7 Update 10修正之后，运行以上代码只能得到如下结果 : i am father



那在新版本的JDK中，上面的问题是否能够得到解决呢？

- 答案是可以的，如果读者去查看MethodHandles.Lookup类的代码，将会发现需要进行哪些访问保护，在该API实现时是预留了后门的。
- 访问保护是通过一个**allowedModes**的参数来控制，而且这个参数可以被设置成“TRUSTED”来绕开所有的保护措施。尽管这个参数只是在Java类库本身使用，没有开放给外部设置，但我们**通过反射可以轻易打破这种限制**。
- 由此，我们可以把代码清单8-16中子类的thinking()方法修改为如下所示的代码来解决问题：

```java
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;
import java.lang.reflect.Field;

/**
 * @author zzm
 */
public class GrandFatherTestCase_2 {

    class GrandFather {
        void thinking() {
            System.out.println("i am grandfather");
        }
    }

    class Father extends GrandFatherTestCase_2.GrandFather {
        void thinking() {
            System.out.println("i am father");
        }
    }

    class Son extends GrandFatherTestCase_2.Father {
        void thinking() {
            try {
                MethodType mt = MethodType.methodType(void.class);
                Field lookupImpl = MethodHandles.Lookup.class.getDeclaredField("IMPL_LOOKUP");
                lookupImpl.setAccessible(true);
                MethodHandle mh = ((MethodHandles.Lookup) lookupImpl.get(null)).findSpecial(GrandFather.class,"thinking", mt, GrandFather.class);
                mh.invoke(this);
            } catch (Throwable e) {
            }
        }

    }

    public static void main(String[] args) {
        (new GrandFatherTestCase_2().new Son()).thinking();
    }

}
```

运行以上代码，在目前所有JDK版本中均可获得如下结果:   i am grandfather



# 