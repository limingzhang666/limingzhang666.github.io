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

（Todo）

## 类的连接阶段

（Todo）

## 类的初始化阶段

（Todo）



好累，脑子一团浆糊了，主要是下雨了，睡觉太舒服了...