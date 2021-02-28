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









## 类的初始化阶段

（Todo）



好累，脑子一团浆糊了，主要是下雨了，睡觉太舒服了...