---
title: 深入理解jvm-06-类文件结构
copyright: true
related_posts: true
date: 2021-01-11 22:03:21
tags: 类文件结构
categories: jvm
---



# 类文件结构

## 无关性的基石

### 平台无关性：

- 各种不同平台的Java虚拟机，以及所有平台都统一支持的程序存储格式——字节码（Byte Code）是构成平台无关性的基石

### 语言无关性：

- Java技术发展之初，设计者们就曾经考虑过并实现了让其他语言运行在Java虚拟机之上的可能性，
- 他们在发布规范文档的时候，也刻意把Java的规范拆分成了：
- ***《Java语言规范》（The Java Language Specification）***
- ***及《Java虚拟机规范》（The Java Virtual Machine Specification）***



***在未来，我们会对Java虚拟机进行适当的扩展，以便更好地支持其他语言运行于Java虚拟机之上”***
***（In the future，we will consider bounded extensions to the Java virtual machine to provide better support for other languages）***



- ***作为一个通用的、与机器无关的执行平台，任何其他语言的实现者都可以将Java虚拟机作为他们语言的运行基础，以Class文件作为他们产品的交付媒介。***

例如，使用Java编译器可以把Java代码编译为存储字节码的Class文件，使用JRuby等其他语言的编译器一样可以把它们的源程序代码编译成Class文件。虚拟机丝毫不关心Class的来源是什么语言

- Java语言中的各种语法、关键字、常量变量和运算符号的语义 最终都会由多条字节码指令组合来表达 ，这决定了字节码指令所能提供的语言描述能力必须比Java语言本身更加强大才行。（类似于 java语言的功能只是 字节码功能的一个子集）
- 因此有一些Java语言本身无法有效支持的语言特性并不代表在字节码中也无法有效表达出来，这为其他程序语言实现一些有别于java的语言特性提供了发挥空间

![](/uploads/jvm/08-JVM-nobindLanguage.png)

## Class类文件的结构

注意：

- 任何一个Class文件都对应着唯一的一个类或接口的定义信息
- 类或接口并不一定都得定义在文件里（譬如类或接口也可以动态生成，直接送入类加载器中）



- Class文件是一组以8个字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在文件之中，中间没有添加任何分隔符，这使得整个Class文件中存储的内容几乎全部是程序运行的必要数据，没有空隙存在
- 当遇到需要占用8个字节以上空间的数据项时，则会按照高位在前[2]的方式分割成若干个8个字节进行存储。



***Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：“无符号数”和“表”***

- 无符号数： 无符号数属于基本的数据类型，以u1、u2、u4、u8来分别代表1个字节、2个字节、4个字节和8个字节的无符号数，无符号数可以用来描述数字、索引引用、数量值或者按照UTF-8编码构成字符串值。
- 表：  表是由多个无符号数或者其他表作为数据项构成的复合数据类型，为了便于区分，所有表的命名都习惯性地以“_info”结尾。表用于描述有层次关系的复合结构的数据，整个Class文件本质上也可以视作是一张表

![](/uploads/jvm/09-jvm-wufahao-info.png)

- 无论是无符号数还是表，当需要描述同一类型但数量不定的多个数据时，经常会使用一个前置的容量计数器加若干个连续的数据项的形式，这时候称这一系列连续的某一类型的数据为某一类型的“集合”。

### 强调：

Class的结构不像XML等描述语言，由于它没有任何分隔符号，无论是顺序还是数量，甚至于数据存储的字节序（Byte Ordering，Class文件中字节序为Big-Endian）这样的细节，都是被严格限定的，哪个字节代表什么含义，长度是多少，先后顺序如何，全部都不允许改变



### 魔数与Class文件的版本

魔数：

- 每个Class文件的头4个字节被称为魔数（Magic Number），它的唯一作用是确定这个文件是否为一个能被虚拟机接受的Class文件

- 使用魔数而不是扩展名来进行识别主要是基于安全考虑，因为文件扩展名可以随意改动
- Class文件的魔数取得很有“浪漫气息”，值为0xCAFEBABE（咖啡宝贝？）
- 紧接着魔数的4个字节存储的是Class文件的版本号：第5和第6个字节是次版本号（MinorVersion），第7和第8个字节是主版本号（Major Version）。Java的版本号是从45开始的，
- 高版本的JDK能向下兼容以前版本的Class文件，但不能运行以后版本的Class文件，因为《Java虚拟机规范》在Class文件校验部分明确要求了即使文件格式并未发生任何变化，虚拟机也必须拒绝执行超过其版本号的Class文件。



### 常量池

#### 定义：

- 紧接着主、次版本号之后的是常量池入口，常量池可以比喻为Class文件里的资源仓库，它是Class文件结构中与其他项目关联最多的数据，通常也是占用Class文件空间最大的数据项目之一，另外，它还是在Class文件中第一个出现的表类型数据项目。

#### 常量池容量计数值

- 由于常量池中常量的数量是不固定的，所以在常量池的入口需要放置一项u2类型的数据，代表常量池容量计数值（constant_pool_count）

- 与Java语言习惯不同，这个容量计数器从1开始，而不是从0开始。

- 之所以从1开始是因为 ：  在Class文件格式规范制定之时，设计者将第0项常量控出来是有特殊考虑，目的在于 ***当某些指向常量池的索引值的数据在特定情况下需要表达“不引用任何一个常量池项目”的含义 ，可以把索引值设置为0***来表示。 

- 如下 图所示，常量池容量（偏移地址：0x00000008）为十六进制数0x0016，即十进制的22，这就代表常量池中有21项常量，索引值范围为1～21。

  ![](/uploads/jvm/10-jvm-constantPool.png)



#### 常量类型：

常量池中主要存放两大类常量：字面量（Literal）和符号引用（Symbolic References）。

- 字面量比较接近于Java语言层面的常量概念，如文本字符串、被声明为final的常量值等。

- 符号引用则属于编译原理方面的概念，主要包括下面几类常量：

  - 被模块导出或者开放的包（Package）
  - 类和接口的全限定名（Fully Qualified Name）
  - 字段的名称和描述符（Descriptor）
  - 方法的名称和描述符
  - 方法句柄和方法类型（Method Handle、Method Type、Invoke Dynamic）
  - 动态调用点和动态常量（Dynamically-Computed Call Site、Dynamically-Computed Constant）

  

#### JVM动态连接

- 在Class文件中不会保存各个方法、字段最终在内存中的布局信息
- 这些字段、方法的符号引用不经过虚拟机在运行期转换的话 是无法得到真正的内存入口地址，也就无法直接被虚拟机直接使用
- 当虚拟机做类加载时，将会从常量池获得对应的符号引用，再在类创建时或运行时解析、翻译到具体的内存地址之中



#### 常量池的项目类型

- 常量池中每一项常量都是一个表，
- 最初常量表中共有11种结构 各不相同的表结构数据，后来为了更好的支持动态语言调用，额外增加了4种动态语言相关的常量（）
  - JDK 7时增加了前三种：**CONSTANT_MethodHandle_info**、**CONSTANT_MethodType_info**和**CONSTANT_InvokeDynamic_info**。出于性能和易用性的考虑（JDK 7设计时已经考虑到，预留了17个常量标志位），在JDK 11中又增加了第四种常量**CONSTANT_Dynamic_info**。
- 为了支持Java模块化系统（Jigsaw），又加入了**CONSTANT_Module_info**和**CONSTANT_Package_info**两个常量
- 截至JDK13，常量表中分别有17种不同类型的常量。

![](/uploads/jvm/12-jvm-constantPool-projectType.png)



#### 常量结构：

- 之所以说常量池时最烦琐的数据，是因为这**17种常量类型各自有着完全独立的数据结构**，两两之间被没有什么共性和联系

##### CONSTANT_Class_info

- 回头看看图6-3中常量池的第一项常量，它的标志位（偏移地址：0x0000000A）是0x07，

- 查表6-3的标志列可知这个常量属于**CONSTANT_Class_info**类型，此类型的常量代表一个**类或者接口的符号引用**。

- CONSTANT_Class_info的结构比较简单，如表6-4所示。

| 类型 | 名称       | 数量 |
| ---- | ---------- | ---- |
| u1   | tag        | 1    |
| u2   | name_index | 1    |

- 其中 tag 是标志位，它用于区分常量类型；
- name_index 是常量池的索引值，它指向常量池中一个 CONSTANT_Utf8_info类型常量，此常量代表了整个类（或者接口）的全限定名。
- 本例中的 name_index值（偏移地址：0x0000000B）为0x0002，也就是指向了常量池中的第二项常量。
- 继续从图6-3中查找第二项常量，它的标志位（地址：0x0000000D）是0x01，查表6-3可知确实是一个**CONSTANT_Utf8_info**类型的常量。

##### CONSTANT_Utf8_info

 CONSTANT_Utf8_info 类型的结构如表6-5所示。
| 类型 | 名称   | 数量   |
| ---- | ------ | ------ |
| u1   | tag    | 1      |
| u2   | length | 1      |
| u1   | bytes  | length |

- length值说明了这个UTF-8编码的字符串长度是多少字节，
- 它后面紧跟着的长度为length字节的连续数据是一个使用UTF-8缩略编码表示的字符串
- UTF-8缩略编码与普通UTF-8编码的区别是：
  从'\u0001'到'\u007f'之间的字符（相当于1～127的ASCII码）的缩略编码使用一个字节表示，
  从'\u0080'到'\u07ff'之间的所有字符的缩略编码用两个字节表示，从'\u0800'开始到'\uffff'之间的所有字符
  的缩略编码就按照普通UTF-8编码规则使用三个字节表示。
- 由于Class文件中方法、字段等都需要引用 CONSTANT_Utf8_info 型常量来描述名称，所以 CONSTANT_Utf8_info型常量的最大长度也就是Java中方法、字段名的最大长度（length的最大值），即u2类型能表达的最大值65535。
- ***所以Java程序中如果定义了超过64KB*** 英文字符的变量或方法名，即使规则和全部字符都是合法的，也会无法编译。



#### javap 工具

在JDK的bin目录中，Oracle公司已经为我们准备好一个专门用于分析Class文件字节码的工具：javap。

代码清单6-2中列出了使用javap工具的-verbose参数输出的TestClass.class文件字节码内容（为节省篇幅，此清单中省略了常量池以外的信息）。



***代码清单6-2　使用javap命令输出常量表***

![](/uploads/jvm/13-jvm-constantPool-javap.png)



- 从代码清单6-2中可以看到，计算机已经帮我们把整个常量池的21项常量都计算了出来
- 其中有些常量似乎从来没有在代码中出现过，如“I”“V”“<init>”“LineNumberTable”“LocalVariableTable”等，这些看起来在源代码中不存在的常量是哪里来的？
- 这部分常量的确不来源于Java源代码，它们都是编译器自动生成的。会被字段表（field_info）、方法表（method_info）、属性表（attribute_info）所引用，它们将会被用来描述一些不方便使用“固定字节”进行表达的内容，譬如描述方法的返回值是什么，有几个参数，每个参数的类型是什么
- 因为Java中的“类”是无穷无尽的，无法通过简单的无符号数来描述一个方法用到了什么类，因此在描述方法的这些信息时，需要引用常量表中的符号引用进行表达



#### 常量池中的17种数据类型的结构总表

![](/uploads/jvm/14-jvm-constantPool-01.png)

![](/uploads/jvm/14-jvm-constantPool-02.png)

![](/uploads/jvm/14-jvm-constantPool-03.png)





### 访问标志（access flags）

在常量池结束之后，紧接着的2个字节代表访问标志（access flags），这个标志用于识别一些类或者接口层次的访问信息。包括：

- 这个Class是类还是接口
- 是否定义为public
- 是否定义为abstract类型
- 如果是类，是否被声明为final
- 等等，具体的标志位以及标志的含义见下：

| 标志名称       | 标志值 | 含义                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 是否为public                                                 |
| ACC_FINAL      | 0x0010 | 是否被声明为final，只有类可以设置                            |
| ACC_SUPER      | 0x0020 | 是否允许使用invokespecial字节码指令的新语义，<br>invokespecial指令的语义在JDK1.0.2发生过改变，为了区别这条指令使用哪种语义，<br>JDK1.0.2之后编译出来的类 这个标志都必须为 真 |
| ACC_INTERFACE  | 0x0200 | 标识这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 是否为abstract类型，对于接口和抽象类来说，此标志为真，其他类型都是 假 |
| ACC_SYNTHETIC  | 0x1000 | 标识 这个类并非由用户代码产生的                              |
| ACC_ANNOTATION | 0x2000 | 标识这是一个注解                                             |
| ACC_ENUM       | 0x4000 | 标识这是一个枚举                                             |
| ACC_MODULE     | 0x8000 | 标识这是一个模块                                             |



### 类索引、父类索引与接口索引集合

- 类索引（this_class）和父类索引（super_class）都是一个u2类型的数据，
- 而接口索引集合（interfaces）是一组u2类型的数据的集合
- Class文件中由这三项数据来确定该类型的继承关系

#### 类索引：

用于确定这个类的全限定名

#### 父类索引：

用于确定这个类的父类的全限定名。由于Java语言不允许多重继承，所以父类索引只有一个，除了java.lang.Object之外，所有的Java类都有父类，因此除了
java.lang.Object外，所有Java类的父类索引都不为0

#### 接口索引集合

用来描述这个类实现了哪些接口，这些被实现的接口将按implements关键字（如果这个Class文件表示的是一个接口，则应当是extends关键字）后的接口顺序从左到右排列在接口索引集合中。



- 类索引、父类索引和接口索引集合都按顺序排列在访问标志之后，类索引和父类索引用两个u2类型的索引值表示，它们各自指向一个类型CONSTANT_Class_info的类描述符常量，

- 通过CONSTANT_Class_info类型的常量中的索引值可以找到定义在CONSTANT_Utf8_info类型的常量中的全限定名字符串

![](/uploads/jvm/15-jvm-class-index.png)



### 字段表(field_info)集合

- 字段表（field_info）用于描述接口或者类中声明的变量。
- Java语言中的“字段”（Field）包括***类级变量以及实例级变量***，
- 但***不包括在方法内部声明的局部变量***

#### 字段可以包括的修饰符有:

字段的作用域（public、private、protected修饰符）、是**实例变量还是类变量（static修饰符）**、可变性（final）、

**并发可见性（volatile修饰符，是否强制从主内存读写**）、**可否被序列化（transient修饰符**）、字段数据类型（基本类型、对象、数组）、字段名称。

- 字段叫做什么名字、字段被定义为什么数据类型，这些都是无法固定的，只能引用常量池中的常量来描述。

#### 字段表的最终格式:

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attribute_info | attributes       | attributes_count |



字段修饰符放在access_flags项目中，它与类中的access_flags项目是非常类似的，都是一个u2的数据类型，其中可以设置的标志位和含义如表6-9所示。

表6-9　字段访问标志
![](/uploads/jvm/16-jvm-attribute-accessFlags.png)

跟随access_flags标志的是两项索引值：name_index和descriptor_index。它们都是对常量池项的引用，分别代表着字段的简单名称以及字段和方法的描述符

#### 全限定名：

以代码清单6-1中的代码为例，“org/fenixsoft/clazz/TestClass”是这个类的全限定名，仅仅是把类全名中的“.”替换成了“/”而已，为了使连续的多个全限定名之间不产生混淆，在使用时最后一般会加入一个“；”号表示全限定名结束。

#### 简单名称：

简单名称则就是指没有类型和参数修饰的方法或者字段名称，这个类中的inc()方法和m字段的简单名称分别就是“inc”和“m”。

#### 描述符：

- 描述符的作用是用来描述字段的数据类型、方法的参数列表（包括数量、类型以及顺序）和返回值。

- 根据描述符规则，基本数据类型（byte、char、double、float、int、long、short、boolean）以及代表无返回值的void类型都用一个大写字符来表示，
- 而对象类型则用字符L加对象的全限定名来表示

![](/uploads/jvm/17-jvm-attribute-desc.png)



- 对于数组类型，每一维度将使用一个前置的“[”字符来描述，如一个定义为“java.lang.String[][]”类型的二维数组将被记录成“[[Ljava/lang/String；”，一个整型数组“int[]”将被记录成“[I”。
- 用描述符来描述方法时，按照先参数列表、后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号“()”之内。如方法void inc()的描述符为“()V”，方法java.lang.String toString()的描述符为“()Ljava/lang/String；”，
- 方法int indexOf(char[]source，int sourceOffset，int sourceCount，char[]target，int targetOffset，int targetCount，int fromIndex)的描述符为“([CII[CIII)I”。

#### PS:

- **字段表集合中不会列出从父类或者父接口中继承而来的字段**

- 但有可能出现原本Java代码之中不存在的字段，譬如在内部类中为了保持对外部类的访问性，编译器就会自动添加指向外部类实例的字
  段。
- 另外，在**Java语言中字段是无法重载的**，两个字段的数据类型、修饰符不管是否相同，都必须使用不一样的名称，
- 但是**对于Class文件格式来讲，只要两个字段的描述符不是完全相同，那字段重名就是合法**的



### 方法表集合

- Class文件存储格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，方法表的结构如同字段表一样，

- 依次包括访问标志（access_flags）、名称索引（name_index）、描述符索引（descriptor_index）、属性表集合（attributes）几项

表6-11　方法表结构

| 类型           | 名称             | 数量             |
| -------------- | ---------------- | ---------------- |
| u2             | access_flags     | 1                |
| u2             | name_index       | 1                |
| u2             | descriptor_index | 1                |
| u2             | attributes_count | 1                |
| attribute_info | attributes       | attributes_count |

- 因为**volatile关键字和transient关键字不能修饰方法**，所以**方法表的访问标志中没有了ACC_VOLATILE标志和ACC_TRANSIENT标志**。
- 与之相对，**synchronized、native、strictfp和abstract关键字可以修饰方法**，**方法表的访问标志中也相应地增加了ACC_SYNCHRONIZED、**
  **ACC_NATIVE、ACC_STRICTFP和ACC_ABSTRACT标志**。
- 对于方法表，所有标志位及其取值可参见表6-12。

![](/uploads/jvm/18-jvm-method_access_flag.png)



#### 方法里的代码去哪了

- 方法的定义可以通过访问标志、名称索引、描述符索引来表达清楚，但方法里面的代码去哪里了
- 方法里的Java代码，经过Javac编译器编译成字节码指令之后，存放在方法属性表集合中一个名为“Code”的属性里面，属性表作为Class文件格式中最具扩展性的一种数据项目，
- 与字段表集合相对应地，如果父类方法在子类中没有被重写（Override），方法表集合中就不会出现来自父类的方法信息。但同样地，有可能会出现由编译器自动添加的方法，最常见的便是类构造器“<clinit>()”方法和实例构造器“<init>()”方法 

#### 重载（ Java语言）

在Java语言中，要重载（Overload）一个方法，除了要与原方法具有相同的简单名称之外，还要求必须拥有一个与原方法不同的特征签名 。

##### 特征签名：

- 指 一个方法中各个参数在常量池中的字段符号引用的集合，也正是因为返回值不会包含在特征签名中，所以java语言里面是无法仅仅依靠返回值的不同来对一个已有方法进行重载的。

- 但是 在Class文件格式之中，特征签名的范围明显要更大一些，只要描述符不是完全一致的两个方法就可以共存。也就是说，如果两个方法有相同的名称和特征签名，但返回值不同，那么也是可以合法共存于同一个Class文件中的
- Java代码的方法特征签名只包括方法名称、参数顺序及参数类型，而字节码的特征签名还包括方法返回值以及受查异常表，

### 属性表（attribute_info）集合



- 与Class文件中其他的数据项目要求严格的顺序、长度和内容不同，属性表集合的限制稍微宽松一些，不再要求各个属性表具有严格顺序，并且《Java虚拟机规范》允许只要不与已有属性名重复，任何人实现的编译器都可以向属性表中写入自己定义的属性信息，Java虚拟机运行时会忽略掉它不认识的属性。
- 为了能正确解析Class文件，《Java虚拟机规范》最初只预定义了9项所有Java虚拟机实现都应当能识别的属性，而在最新的《Java虚拟机规范》的Java SE 12版本中，预定义属性已经增加到29项，这些属性具体见表6-13。

表6-13　虚拟机规范预定义的属性

![](/uploads/jvm/19-jvm-attributeinfo-01.png)

![](/uploads/jvm/19-jvm-attributeinfo-02.png)



#### Code属性：

- Java程序方法体里面的代码经过Javac编译器处理之后，最终变为字节码指令存储在Code属性内。

- Code属性出现在方法表的属性集合之中，但并非所有的方法表都必须存在这个属性，

- 譬如接口或者抽象类中的方法就不存在Code属性，如果方法表有Code属性存在，那么它的结构将如表6-15所示。

表6-15　Code属性表的结构

![](/uploads/jvm/20-jvm-attributeinfo-Code.png)

- attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，此常量值固定为“Code”，它代表了该属性的属性名称，

- attribute_length指示了属性值的长度，由于属性名称索引与属性长度一共为6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。
- max_stack代表了操作数栈（Operand Stack）深度的最大值。在方法执行的任意时刻，操作数栈都不会超过这个深度，虚拟机运行的时候需要根据这个值来分配栈帧（Stack Frame）中的操作栈深度
- max_locals代表了局部变量表所需的存储空间。在这里，max_locals的单位是变量槽（Slot），变量槽是虚拟机为局部变量分配内存所使用的最小单位。对于byte、char、float、int、short、boolean和returnAddress等长度不超过32位的数据类型，每个局部变量占用一个变量槽，而double和long这两种64位的数据类型则需要两个变量槽来存放。
- 方法参数（包括实例方法中的隐藏参数“this”）、显式异常处理程序的参数（Exception Handler Parameter，就是try-catch语句中catch块中所定义的异常）、方法体中定义的局部变量都需要依赖局部变量表来存放。操作数栈和局部变量表直接决定一个该方法的栈帧所耗费的内存，不必要的操作数栈深度和变量槽数量会造成内存的浪费。Java虚拟机的做法是将局部变量表中的变量槽进行重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占的变量槽可以被其他局部变量所使用，Javac编译器会根据变量的作用域来分配变量槽给各个变量使用，根据同时生存的最大局部变量数量和类型计算出max_locals的大小。
- code_length和code用来存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code是用于存储字节码指令的一系列字节流。既然叫字节码指令，那顾名思义每个指令就是一个u1类型的单字节，当虚拟机读取到code中的一个字节码时，就可以对应找出这个字节码代表的是什么指令，并且可以知道这条指令后面是否需要跟随参数，以及后续的参数应当如何解析。我们知道一个u1数据类型的取值范围为0x00～0xFF，对应十进制的0～255，也就是一共可以表达256条指令。
- ***ps:***   关于code_length，有一件值得注意的事情，虽然它是一个u4类型的长度值，理论上最大值可以达到2的32次幂，但是《Java虚拟机规范》中明确限制了***一个方法不允许超过65535条字节码指令***，即它实际只使用了u2的长度，***如果超过这个限制，Javac编译器就会拒绝编译***。一般来讲，编写Java代码时只要不是刻意去编写一个超级长的方法来为难编译器，是不太可能超过这个最大值的限制的。但是，某些特殊情况，例如在编译一个很复杂的JSP文件时，某些JSP编译器会把JSP内容和页面输出的信息归并于一个方法之中，就有可能因为方法生成字节码超长的原因而导致编译失败。





如果***把一个Java程序中的信息分为代码（Code，方法体里面的Java代码）和元数据（Metadata，包括类、字段、方法定义及其他信息）两部分***，那么在整
个Class文件里，**Code属性用于描述代码**，**所有的其他数据项目都用于描述元数据**。





```bash
// 原始Java代码
public class TestClass {
    private int m;
    public int inc() {
    	return m + 1;
    }
}
///////////////////编译后


C:\>javap -verbose TestClass
// 常量表部分的输出见代码清单6-1，因版面原因这里省略掉
{
public org.fenixsoft.clazz.TestClass();
//构造方法
    Code:
        Stack=1, Locals=1, Args_size=1
        0: aload_0
        1: invokespecial #10; //Method java/lang/Object."<init>":()V
        4: return
    LineNumberTable:
    	line 3: 0
    LocalVariableTable:
        Start Length Slot Name Signature
        0 5 0 this Lorg/fenixsoft/clazz/TestClass;

// 方法 2
public int inc();
Code:
    Stack=2, Locals=1, Args_size=1
    0: aload_0
    1: getfield #18; //Field m:I
    4: iconst_1
    5: iadd
    6: ireturn
LineNumberTable:
	line 8: 0
LocalVariableTable:
    Start Length Slot Name Signature
    0 7 0 this Lorg/fenixsoft/clazz/TestClass;
}
```



#### 疑问:

- 这个类有两个方法——实例构造器<init>()和inc()，这两个方法很明显都是没有参数的，为什么Args_size会为1？
- 而且无论是在参数列表里还是方法体内，都没有定义任何局部变量，那Locals又为什么会等于1？

#### 答案：

- 一条Java语言里面的潜规则：在任何实例方法里面，都可以通过“this”关键字访问到此方法所属的对象。这个访问机制对Java程序的编写很重要，

- 而它的实现非常简单，仅仅是通过在Javac编译器编译的时候把对this关键字的访问转变为对一个普通方法参数的访问，然后在虚拟机调用实例方法时自动传入此参数而已。
- 因此在实例方法的局部变量表中至少会存在一个指向当前对象实例的局部变量，局部变量表中也会预留出第一个变量槽位来存放对象实例的引用，所以实例方法参数值从1开始计算。
- 这个处理只对实例方法有效，如果代码清单6-1中的inc()方法被声明为static，那Args_size就不会等于1而是等于0了。
- 

### 异常表：

如果存在异常表，那它的格式应如表6-16所示，包含四个字段，这些字段的含义为：如果当字节码从第start_pc行[1]到第end_pc行之间（不含第end_pc行）出现了类型为catch_type或者其子类的异常（catch_type为指向一个CONSTANT_Class_info型常量的索引），则转到第handler_pc行继续处理。当catch_type的值为0时，代表任意异常情况都需要转到handler_pc处进行处理。

表6-16　属性表结构

| 类型 | 名称       | 数量 |
| ---- | ---------- | ---- |
| u2   | start_pc   | 1    |
| u2   | end_pc     | 1    |
| u2   | handler_pc | 1    |
| u2   | catch_pc   | 1    |

- 异常表实际上是Java代码的一部分，尽管字节码中有最初为处理异常而设计的跳转指令，
- 但《Java虚拟机规范》中明确要求Java语言的编译器**应当选择使用异常表**而不是通过跳转指令来实现Java异常及finally处理机制[2]。

代码清单6-5是一段演示异常表如何运作的例子，这段代码主要演示了**在字节码层面try-catchfinally是如何体现的**

```bash\
// Java源码
public int inc() {
    int x;
    try {
        x = 1;
    	return x;
    } catch (Exception e) {
    	x = 2;
   	 	return x;
    } finally {
    	x = 3;
    }
}
// 编译后的ByteCode字节码及异常表
public int inc();
    Code:
        Stack=1, Locals=5, Args_size=1
            0: iconst_1 // try块中的x=1
            1: istore_1
            2: iload_1 // 保存x到returnValue中，此时x=1
            3: istore 4
            5: iconst_3 // finaly块中的x=3
            6: istore_1
            7: iload 4 // 将returnValue中的值放到栈顶，准备给ireturn返回
            9: ireturn
            10: astore_2 // 给catch中定义的Exception e赋值，存储在变量槽 2中
            11: iconst_2 // catch块中的x=2
            12: istore_1
            13: iload_1 // 保存x到returnValue中，此时x=2
            14: istore 4
            16: iconst_3 // finaly块中的x=3
            17: istore_1
            18: iload 4 // 将returnValue中的值放到栈顶，准备给ireturn返回
            20: ireturn
            21: astore_3 // 如果出现了不属于java.lang.Exception及其子类的异常才会走到这里
            22: iconst_3 // finaly块中的x=3
            23: istore_1
            24: aload_3 // 将异常放置到栈顶，并抛出
            25: athrow
    Exception table:
    from to target type
        0 5 10 Class java/lang/Exception
        0 5 21 any
        10 16 21 any

```



编译器为这段Java源码生成了三条异常表记录，对应三条可能出现的代码执行路径。从Java代码的语义上讲，这三条执行路径分别为：

- 如果try语句块中出现属于Exception或其子类的异常，转到catch语句块处理；
- 如果try语句块中出现不属于Exception或其子类的异常，转到finally语句块处理；
- 如果catch语句块中出现任何异常，转到finally语句块处理。



返回到我们上面提出的问题，这段代码的返回值应该是多少？熟悉Java语言的读者应该很容易说出答案：

- 如果没有出现异常，返回值是1；
- 如果出现了Exception异常，返回值是2；
- 如果出现了Exception以外的异常，方法非正常退出，没有返回值。

我们一起来分析一下字节码的执行过程，从字节码的层面上看看为何会有这样的返回结果。

1. 字节码中第0～4行所做的操作就是将整数1赋值给变量x，并且将此时x的值复制一份副本到最后一
   个本地变量表的变量槽中（这个变量槽里面的值在ireturn指令执行前将会被重新读到操作栈顶，作为
   方法返回值使用。为了讲解方便，给这个变量槽起个名字：returnValue）。
2. 如果这时候没有出现异常，则会继续走到第5～9行，将变量x赋值为3，然后将之前保存在returnValue中的整数1读入到操作栈
   顶，最后ireturn指令会以int形式返回操作栈顶中的值，方法结束。
3. 如果出现了异常，PC寄存器指针转到第10行，第10～20行所做的事情是将2赋值给变量x，然后将变量x此时的值赋给returnValue，最后再
   将变量x的值改为3。方法返回前同样将returnValue中保留的整数2读到了操作栈顶。从第21行开始的代码，作用是将变量x的值赋为3，并将栈顶的异常抛出，方法结束。





TODO ： 其他属性表 集合 && 字节码指令集