---
title: java高并发详解-09-类的加载过程
copyright: true
comments: true
related_posts: true
date: 2021-02-25 22:29:27
tags: 类的加载过程
categories: 高并发详解
---

- ClassLoader的主要职责是负责加载各种class文件到JVM中
- ClassLoader是一个抽象的class ，给定一个class 的二进制文件名，ClassLoader会尝试加载并且在JVM中生成构成这个类的各个数据结构，然后使其分布在JVM对应的内存区域中

# 类的加载过程简介

类的加载过程一般分为3个比较大的阶段，分别是 **加载阶段、连接阶段、初始化阶段，**

![](/uploads/java-concurrency-master/loadClassStep.png)

- 加载阶段： 主要负责查找并加载类的二进制数据文件，其实就是class文件
- 连接阶段： 这个阶段所做的工作比较多，细分的话还可以分为以下3个阶段
  - 验证： 主要是确保类文件的正确性，比如class的版本，class文件的魔术因子是否正确
  - 准备： 为类的静态变量分配内存，并且为其初始化默认值
  - 解析： 把类中的符号引用转换为 直接引用
- 初始化阶段： 为类的静态变量赋予正确的初始值 （代码编写阶段给定的值，也就是程序员的代码赋的值）



当一个JVM在我们通过执行Java 命令启动后，其中可能包含的类非常多，并不是每个类都会被初始化。

- JVM对类的初始化时一个延迟的机制，即 ：使用的时 lazy的方式，当一个类在首次使用的时候才会被 初始化，同一个运行时包下，一个Class 只会被初始化一次 （运行时包和类的包是有区别的）



# 类的主动使用和被动使用

jvm虚拟机规范规定了，每个类或者接口被Java程序首次主动使用时，才会对其进行初始化。

## 主动使用

(下面的例子有问题，main方法应该写在别的类里面，因为main方法会导致类初始化)

jvm同时规范了以下6种主动使用类的场景，具体如下

1. 通过new 关键字会导致类的初始化： 这种是我们经常采用的初始化一个类的方式，它肯定会导致类的加载并且最终初始化。

2. **访问类的静态变量**，包括读取和更新会导致类的初始化

   ```java
   public class Simple {
       static {
           System.out.println("I will be initialized");
       }
   
       public static int x = 10;
   
      
   }
   
   public class Test{
         public static void main(String[] args) throws ClassNotFoundException {
           // Simple[] simples = new Simple[10];
           System.out.println(x);
       }
   }
   运行结果：
       I will be initialized
   	10
   ```

   3. **访问类的静态方法**，会导致类的初始化 

   ```java
   public class Simple {
       static {
           System.out.println("I will be initialized");
       }
   
       public static int x = 10;
       public static void test(){
           System.out.println("test execute");
       }
   
      
   }
   public class Test{
        public static void main(String[] args) throws ClassNotFoundException {
           // Simple[] simples = new Simple[10];
          // System.out.println(x);
           test();
       }
   }
   
   运行结果：
       I will be initialized
   test execute
   ```

   4. 对某个类进行反射操作，会导致类的初始化 

      ```java
          public static void main(String[] args) throws ClassNotFoundException {
              // Simple[] simples = new Simple[10];
             // System.out.println(x);
      //        test();
              Class.forName("com.wangwenjun.concurrent.chapter09.Simple");
          }
      
      运行结果:
      I will be initialized
      ```

      5. 初始化子类会导致父类的初始化 

         ```java
         //父类
         public class Parent {
             static {
                 System.out.println("The parent is initialized");
             }
         
             public static int y = 100;
         }
         // 子类
         public class Child extends Parent {
             static {
                 System.out.println("The child will be initialized");
             }
         
             public static int x = 10;
         
             public int test() {
                 return 0;
             }
         
             public void test(int x) {
             }
         }
         //测试类
         public class ActiveLoadTest {
             static {
                 System.out.println("ActiveLoadTest test");
             }
         
             public static void main(String[] args) {
                 /*Simple[] simples = new Simple[10];
                 System.out.println(simples.length);*/
         
         //        System.out.println(GlobalConstants.MAX);
         //        System.out.println(GlobalConstants.RANDOM);
                 System.out.println(Child.x);
             }
         }
         运行结果：
             ActiveLoadTest test
         The parent is initialized
         The child will be initialized
         10
         
         ```

         - 在测试类中调用了Child的静态变量，会使得Child被初始化，Child又是 Parent的子类， 子类的初始化会导致父类的初始化

         - 需要注意的一点是，如果**通过子类使用父类的 静态变量（输出Child.y）只会导致父类的初始化，子类则不会被初始化**

           ```java
           //测试类
           public class ActiveLoadTest {
               static {
                   System.out.println("ActiveLoadTest test");
               }
           
               public static void main(String[] args) {
                   /*Simple[] simples = new Simple[10];
                   System.out.println(simples.length);*/
           
           //        System.out.println(GlobalConstants.MAX);
           //        System.out.println(GlobalConstants.RANDOM);
                   System.out.println(Child.x);
               }
           }
           运行结果：
               
           ActiveLoadTest test
           The parent is initialized
           100
           ```

           6. 启动类： 也就是执行main函数所在的类会导致该类的初始化， 比如使用java命令运行上文中的ActiveLoadTest类



## 被动使用

除了上面的6种情况，其余的都称为被动使用，不会导致类的加载和初始化

1. 构造某个类的数组时，并不会导致该类的初始化

   ```java
   public class ActiveLoadTest {
       static {
           System.out.println("ActiveLoadTest test");
       }
   
       public static void main(String[] args) {
           /*Simple[] simples = new Simple[10];
           System.out.println(simples.length);*/
   
   //        System.out.println(GlobalConstants.MAX);
   //        System.out.println(GlobalConstants.RANDOM);
   //        System.out.println(Child.y);
   
           Simple[] simples = new Simple[10];
   //        System.out.println(simples);
           // System.out.println(x);
   //            test();
   //        Class.forName("com.wangwenjun.concurrent.chapter09.Simple");
       }
   }
   ```

   

2.  引用**类的静态常量**不会导致类的初始化

```java
public class GlobalConstants {
    static {
        System.out.println("The GlobalConstants will be initialized.");
    }

    // 在其他类中使用 MAX 不会导致 GlobalConstants的初始化，静态代码块不会输出
    public final static int MAX = 100;
    // 虽然 RANDOM 是静态常量，但是由于计算复杂，只有初始化之后才能得到记过，因此在其他类中使用 RANDOM 会导致 GlobalConstants的初始化
    public final static int RANDOM = new Random().nextInt();
}

 public static void main(String[] args) {
        /*Simple[] simples = new Simple[10];
        System.out.println(simples.length);*/

        System.out.println(GlobalConstants.MAX);
        System.out.println(GlobalConstants.RANDOM);
//        System.out.println(Child.y);
//
//        Simple[] simples = new Simple[10];
//        System.out.println(simples);
        // System.out.println(x);
//            test();
//        Class.forName("com.wangwenjun.concurrent.chapter09.Simple");
    }

运行结果：
    ActiveLoadTest test
100
The GlobalConstants will be initialized.
1357034692
```



# 类的加载过程详解

## 类的加载阶段

类的加载就是将class文件中的二进制数据读取到内存之中，然后将该字节流所代表的静态存储结构转换为方法区中 运行的数据结构，并且在堆内存中生成一个该类的 java.lang.Class 对象，作为访问方法区数据结构的入口。

![](/uploads/java-concurrency-master/afterCLassLoader.png)

- 类加载的最终产物就是堆内存中的class对象，对同一个ClassLoader来讲，不管某个类被加载了多少次，对应到堆内存中的class对象始终是同一个。
- 虚拟机规范中指出了类的加载是通过一个**全限定名（包名+类名）**来获取二进制数据流，
- 但是并没有限定必须通过某种方式去获得，例如常见的是  **class二进制文件的形式**
  - 运行时动态生成，比如通过动态代理 java.lang.Proxy也可以生成代理类的二进制字节流
  - 通过网络获取
  - 通过读取zip文件获得类的二进制字节流，比如jar 、war
  - 将类的二进制数据存储在数据库的BLOB字段类型中
  - 运行时生成class文件，并且动态加载
- 在某个类完成加载阶段之后，虚拟机会**将这些二进制字节流按照虚拟机所需的格式存储在*方法区*中**，然后形成特定的数据结构，随之**又在堆内存中实例化一个 java.lang.Class类对象，**在类加载的整个生命周期中，加载过程还没有结束，连接阶段是可以交叉工作的，比如连接阶段验证字节流信息的合法性。

## 类的连接阶段

类的连接阶段可以细分为3个小的过程，分别是**验证、准备和解析**

### 1.验证

验证在连接阶段的主要目的是：  **确保class文件的字节流所包含的内存符合当前JVM的规范要求，并且不会危害JVM 自身安全的代码**，当字节流信息不符合要求时，则会抛出 VerifyError 这样的异常或者时子异常。

- 验证文件格式（例如魔术因子 0xCAFEBABE, 主次版本号）
- 元数据的验证
  - 元数据的验证其实是对class的字节流进行语义分析的过程，确保class字节流符合JVM规范的要求
  - 检查当前类是否存在父类，是否继承了某个接口，这些父类和接口是否合法
  - 检查该类是否继承了被final修饰的类，被final修饰的类是不允许被继承并且其中的方法是不允许被 override的
  - 检查该类是否为抽象类，如果不是抽象类，是否实现了父类的抽象方法或者接口中的所有方法
  - 检查方法重载的合法性，比如相同的方法名称、相同的参数，但是返回类型不同，这都是不被允许的
- 字节码验证
  - 主要是验证程序的控制流程，比如循环、分支。比如： 类型转换是否合法 ，程序计数器的指令不会跳转到不合法的字节码指令中去
- 符号引用的验证
  - 验证符号引用转换为直接引用时的合法性
  - 通过符号引用描述的字符串全限定名称是否能够顺利的找到相关的类
  - 符号引用的类、字段、方法，是否能对当前类可见，比如不能访问引用类的私有方法
  - 符号引用验证的目的是： 为了保证解析动作的顺利进行，比如某个类的字段不存在，则会抛出NoSuchFieldError ，若方法不存在 则抛出 NoSuchMethodError等，我们在使用反射的时候，会遇到这样的异常信息



### 2.准备

当一个class字节流通过了所有的验证过程后，就开始为该对象的类变量（静态变量）分配内存并设置初始值了。 

注意： **类变量的内存会被分配到方法区中，** 不同于实例变量会被分配到对内存中 。



- 所谓的设置初始值，其实就是为相应的类变量 给定一个相关类型在没有设置值时的默认值，不同的数据类型以及其 初始值为：

| 数据类型 | 初始值    |
| -------- | --------- |
| Byte     | （byte）0 |
| Char     | ‘\u0000’  |
| Short    | (short)0  |
| Int      | 0         |
| Float    | 0.0F      |
| Double   | 0.0D      |
| Long     | 0L        |
| Boolean  | False     |
| 引用类型 | null      |



```java
public class LinkedPrepate{
    private static int a=10;
    private final static int b=10;
}
```

- static int a=10,在准备阶段值为 初始值0，
- final static int b则为10 ，因为在类的编译阶段javac 会将其Value 生成一个 ConstantValue属性，直接赋予10.



### 3.解析（TODO）

在连接阶段中经历了验证、准备之后，就可以顺利进入到解析过程了，当然在解析的过程中照样会交叉一些验证的过程，

- 比如符号引用的验证，

所谓解析就是在常量池中寻找类、接口、字段和方法的符号引用，并且将这些符号引用替换为直接引用的过程：

```java


public class ClassResolve{
    static Simple simple=new Simple();
    pulbic static void main(String[] args){
        System.out.println(simple);
    }
}
```

虚拟机规范规定了，在anewarray，checkcast、getfield、getstatic，instanceof，invokeinterface，invokespecial、invokevirtual，multianewarray、new、putfiled、putstatic 这13个 操作符号引用的字节码指令之前，必须要对所有的符号提前进行解析 。



解析过程主要是 针对类接口、字段、类方法和接口方法这4类进行的，分别对应到常量池中的CONSTANT_Class_info、 Constant_Filldref_info、Constant_Methodref_info 和 Constant_InterfaceMethodref_info 这4中类型常量 。

#### 类接口解析

- 假设前文代码中的Simple ，不是一个数组类型，则在加载的过程中，需要先完成对SImple 类的加载，同样需要经历所有的类加载阶段
- 如果SImple是一个数组类型，则虚拟机不需要完成对 SImple 的加载，只需要在虚拟机中生成一个能够代表该类型的数组对象，并且在堆内存中开辟一片连续的地址空间即可
- 在类接口的解析完成之后，还需要进行符号引用的验证

#### 字段的解析

所谓字段的解析，就是解析你所访问类或者接口中的字段，在解析类或者变量的时候，如果该字段不存在，或者出现错误，则会抛出异常，不再进行下面的解析。

- 如果Simple类本身就包含某个字段，则直接返回这个字段的引用，当然也要对该字段所属的类提前进行类加载
- 如果Simple 类中不存在该字段，则会根据继承关系自下而上，查找父类或者接口的字段，找到即可返回，同样需要提前对找到的字段进行类的加载过程 。
- 如果SImple类中没有字段，一直找到了最上层的java.lang.Object 还是没有，则表示 查找失败，也就不再进行任何解析，直接抛出了NoSuchFieldError 异常

#### 类方法的解析

类方法和接口方法有所不同，类方法可以直接使用该类进行调用，而接口方法必须要有相应的实现类继承才能够进行调用 。

- 若在类方法表中发现class_index 中索引的Simple 是一个接口而不是一个类，则 直接返回错误
- 在SImple类中查找是否有方法描述和目标方法完全一致的方法，如果有，则直接返回这个方法的引用，否则直接继续向上查找。
- 如果父类中仍然没有找到，则意味着查找失败，程序会抛出NoSuchMethodError 异常
- 如果在当前类或者父类中找到了和目标方法一致的方法，但是它是一个抽象类，则会抛出AbstractMethodError 这个异常

#### 接口方法的解析

接口不仅可以定义方法，还可以继承其他接口

- 在接口方法表中发现 class_index 中索引的Simple是一个类而不是一个接口，则会直接 返回错误，因为方法接口表 和类接口表 所容纳的类型应该是 不一样的，所以常量池 中有 Constant_Methodref_info 和 Constant_InterfaceMethodref_info 两个不同的类型 
- 接下来的查找 和类方法的解析就比较类似了，自下而上的查找，直到找到为止，或者没找到 抛出NoSuchMethodError 异常 。

 

### 类的初始化阶段 

类的初始化阶段是整个类加载过程的最后一个阶段

- 在初始化阶段做的最主要的一件事情就是执行 <clinit> () 方法中所有的类变量都会被 赋予正确的值，也就是在程序编写的时候指定的值 
-  <clinit> () 方法 实在编译阶段生成的，也就是说它 已经包含在 class文件中了，<clinit>中包含了所有类变量的赋值动作和静态语句块的执行代码 ，
- 编译器收集的顺序是 由执行语句在源文件中的出现顺序所决定的 （<clinit> 是能够保证顺序性）
- 静态 语句块只能对后面的静态变量进行赋值，但是 不能对其进行访问。

```java
public class ClassInit{
    
     static{
         // 静态代码块 只能对后面的静态变量进行赋值，但是不能对其访问 
        System.out.println(x);// 报错，Illegal forward reference 
        //  
        x=100;
    }
    private static int x=10;
    
}
```

- 另外<clinit> 方法与类的构造函数有所不同，它不需要显示的调用父类的 构造器，虚拟机会保证父类的 <clinit>方法最先执行，因此父类的静态变量总是能够得到优先赋值

```java
public class ClassInit{
    static class Parent {
        static int value = 10;

        static {
            value = 20;
        }
    }

    // 子类使用父类的静态变量为自己的静态变量赋值
    static class Child extends Parent {
        static int i = value;
    }

    public static void main(String[] args) {
        System.out.println(Child.i);
    }
}

输出的是： 20 
```

- 虚拟机会保证父类的 <clinit>方法最先执行，因此父类的静态变量总是 能够得到优先赋值  
- <clinit>() 方法虽然是真实存在的，但是它 只能被虚拟机执行，在主动使用触发了类的初始化之后就会调用这个方法  

```java
public class ClassInit{
    
    static {
        try {
            System.out.println("The ClassInit static code block will be invoke.");
            TimeUnit.SECONDS.sleep(10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        IntStream.range(0, 5)
                .forEach(i -> new Thread(ClassInit::new));
    }
}
```

- 在同一时间，只能有一个线程执行到静态代码块中的内容，并且静态代码块仅仅只会被执行一次，JVM保证了 <clinit> 方法在多线程的执行环境下的同步语义，因此在单例设计模式下，采用 Holder的方式是一种最佳的设计方法 



# 本章总结

- 类的加载过程 还是会围绕 二进制文件的加载，二进制数据的连接以及类的初始化这样的过程区进行 

```java
public class Singleton{
   //1. 
    public static int x=0;
    private static int y;
    private static Singleton instance=new Singleton(); // 2.
    private Singleton(){
        x++;
        y+;
    }
    public static Singleton getInstance(){
        return instance;
    }
    
    public static void main(String[] args){
        Singleton singleton=Singleton.getInstance();
        System.out.println(singleton.x);
        System.out.println(singleton.y);
    }
    
}

运行结果：
    com.wangwenjun.concurrent.chapter09.Singleton@4554617c
0
1
```

1. 首先在连接阶段的准备过程中，每一个类变量都被赋予了相应的初始值  x=0，y=0， instance=null

2. 类的初始化阶段，初始化阶段会为每一个类变量赋予正确的值，也就是执行 <clinit> 方法的过程 

   x=0, y=0, instance =new Singleton()

3.  然后在 new SIngleton 的时候，会执行类的构造函数，而在构造函数中 分别对 x和y 进行自增，结果为：

   x=1，  y=1



再看调换 

```java
public class Singleton {

    private static int x = 0;
    private static int y;
    private static Singleton instance = new Singleton(); // 调换Singleton顺序之后， 
    private Singleton() {
        x++;
        y++;
    }

    public static Singleton getInstance() {
        return instance;
    }

    public static void main(String[] args) {
        Singleton singleton = Singleton.getInstance();
        System.out.println(singleton);
        System.out.println(singleton.x);
        System.out.println(singleton.y);
    }
}
运行结果：
    
    com.wangwenjun.concurrent.chapter09.Singleton@4554617c
0
1
```

