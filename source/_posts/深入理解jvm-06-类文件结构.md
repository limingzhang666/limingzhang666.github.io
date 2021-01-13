---
title:  深入理解jvm-06-类文件结构
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

#### 异常表：

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
            3: istore   4
            5: iconst_3 // finaly块中的x=3
            6: istore_1
            7: iload     4 // 将returnValue中的值放到栈顶，准备给ireturn返回
            9: ireturn
            10: astore_2 // 给catch中定义的Exception e赋值，存储在变量槽 2中
            11: iconst_2 // catch块中的x=2
            12: istore_1
            13: iload_1 // 保存x到returnValue中，此时x=2
            14: istore    4
            16: iconst_3 // finaly块中的x=3
            17: istore_1
            18: iload      4 // 将returnValue中的值放到栈顶，准备给ireturn返回
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

1. 字节码中第0～4行所做的操作就是将整数1赋值给变量x，并且***将此时x的值复制一份副本到最后一***
   ***个本地变量表的变量槽中***（***这个变量槽里面的值在ireturn指令执行前将会被重新读到操作栈顶，作为***
   ***方法返回值使用***。为了讲解方便，给这个变量槽起个名字：returnValue）。
2. 如果这时候没有出现异常，则会继续走到第5～9行，将变量x赋值为3，然后***将之前保存在returnValue中的整数1***  读入到 ***操作栈***
   ***顶***，***最后ireturn指令会以int形式返回操作栈顶中的值***，方法结束。
3. 如果出现了异常，PC寄存器指针转到第10行，第10～20行所做的事情是**将2赋值给变量x**，然后将变量***x此时的值赋给returnValue***，最后**再**
   **将变量x的值改为3**。***方法返回前同样将returnValue中保留的整数2读到了操作栈顶***。
4. 从第21行开始的代码，作用是**将变量x的值赋为3**，并**将栈顶的异常抛出**，方法结束。

***总结一下：***

- 如果先给x赋值，再将x值 复制一个副本到本地变量表的 变量槽中，该变量槽中的值在ireturn指令执行时 会被读到操作栈顶，就是最终要 return的值，而不是 x的变量值
- 如果没有异常，还是会走finally的方法块给x进行赋值，但是最终返回的是 之前本地变量表的变量槽中的值，并不是走完finally块的时候 x的值 
- 如果发生了异常，且异常属于 catch方法的异常或者其异常子类，则 进catch方法块，给x赋值为2，然后将x=2 的值赋给returnValue，然后执行finally 的x=3,最后将 returnValue的值读到栈顶，输出
- 如果发生了catch块之外的异常，则 先执行finally 的x=3，再将异常堆栈信息读到栈顶，抛出去，最终不会return x，而是以异常堆栈的形式结束

```bash
public class Test2 {
    // Java源码
    public int inc() {
        int x;
        try {
            x = 1;
            List<Object> list = Arrays.asList();
//            list.get(1);
            System.out.println(1 / 0);
            System.out.println("异常");

            return x;
//            ClassCastException("异常外")
        } catch (Exception e) {
            System.out.println("catch");
            x = 2;
            return x;
        } finally {
            System.out.println("finally");
            x = 3;
//            return x;
        }
    }

    public static void main(String[] args) {
        Test2 test2 = new Test2();
        int inc = test2.inc();
        System.out.println(inc);
    }
}
```







#### Exceptions属性

- 这里的Exceptions属性是在方法表中与Code属性平级的一项属性，不要与前面刚刚讲解完的异常表产生混淆。

- Exceptions属性的作用是列举出方法中可能抛出的受查异常（Checked Excepitons），也就是方法描述时在throws关键字后面列举的异常。

  它的结构见表6-17。

| 类型 | 名称                  | 数量                |
| ---- | --------------------- | ------------------- |
| u2   | attribute_name_index  | 1                   |
| u4   | attribute_length      | 1                   |
| u2   | number_of_exceptions  | 1                   |
| u2   | exception_index_table | number_of_exceptons |

此属性中的number_of_exceptions项表示方法可能抛出number_of_exceptions种受查异常，每一种受查异常使用一个exception_index_table项表示；exception_index_table是一个指向常量池中CONSTANT_Class_info型常量的索引，代表了该受查异常的类型。



#### LocalVariableTable及LocalVariableTypeTable属性

- LocalVariableTable属性用于描述***栈帧中局部变量表的变量与Java源码中定义的变量之间的关系***，

- 它也不是运行时必需的属性，但默认会生成到Class文件之中，可以在Javac中使用-g：none或-g：vars选项来取消或要求生成这项信息。
- 如果没有生成这项属性，最大的影响就是当其他人引用这个方法时，所有的参数名称都将会丢失，譬如IDE将会使用诸如arg0、arg1之类的占位符代替原有的参数名，这对程序运行没有影响，但是会对代码编写带来较大不便，而且在调试期间无法根据参数名称从上下文中获得参数值。
- LocalVariableTable属性的结构如表6-19所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 | local_variable_table_length | 1                   |
| loca_variable_info |       loca_variable_table                     |local_variable_table_length |

其中local_variable_info项目代表了一个栈帧与源码中的局部变量的关联，结构如表6-20所示。
| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 |start_pc       | 1                   |
| u2                 | length           | 1                   |
| u2                 |name_index | 1                   |
| u2                 | descriptor_index                     |1 |
| u2                 | index | 1                   |

- start_pc和length属性分别代表了这个局部变量的生命周期开始的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围
- name_index和descriptor_index都是指向常量池中CONSTANT_Utf8_info型常量的索引，分别代表了局部变量的名称以及这个局部变量的描述符。
- index是这个局部变量在栈帧的局部变量表中变量槽的位置。当这个变量数据类型是64位类型时（double和long），它占用的变量槽为index和index+1两个

#### LocalVariableTypeTable（用于支持泛型）

- 在JDK 5引入泛型之后，LocalVariableTable属性增加了一个“姐妹属性”——LocalVariableTypeTable。这个新增的属性结构与LocalVariableTable非常相似，仅仅是把记录的字段描述符的descriptor_index替换成了字段的特征签名（Signature）。

- 对于非泛型类型来说，描述符和特征签名能描述的信息是能吻合一致的，但是泛型引入之后，由于描述符中泛型的参数化类型被擦除掉[3]，描述符就不能准确描述泛型类型了。因此出现了LocalVariableTypeTable属性，使用字段的特征签名来完成泛型的描述。

#### ConstantValue属性

- ConstantValue属性的作用是通知虚拟机自动为静态变量赋值。只有被static关键字修饰的变量（类变量）才可以使用这项属性。

- 类似“int x=123”和“static int x=123”这样的变量定义在Java程序里面是非常常见的事情，但虚拟机对这两种变量赋值的方式和时刻都有所不同。
- 对***非static类型的变量（也就是实例变量）的赋值是在实例构造器<init>()方法中进行的***；

- 而对于***类变量***，则有两种方式可以选择：
- - 在类构造器<clinit>()方法中
  - 或者使用ConstantValue属性。

- 目前Oracle公司实现的Javac编译器的选择是，如果***同时使用final和static来修饰一个变量***（按照习惯，这里称“常量”更贴切），并且这个变量的**数据类型是基本类型或者java.lang.String**的话，就将会**生成ConstantValue属性**来进行初始化；

- 如果这个变量**没有被final修饰**，或者**并非基本类型及字符串**，则将会选择在<clinit>()方法中进行初始化。
- 虽然有final关键字才更符合“ConstantValue”的语义，但**《Java虚拟机规范》中并没有强制要求字段必须设置ACC_FINAL标志，只要求有ConstantValue属性的字段必须设置ACC_STATIC标志而已，**

- **对final关键字的要求是Javac编译器自己加入的限制**。而对ConstantValue的属性值只能限于基本类型和String这点，其实并不能算是什么限制，这是理所当然的结果。

因为此属性的属性值只是一个常量池的索引号，***由于Class文件格式的常量类型中只有与基本属性和字符串相对应的字面量***，所以就算
ConstantValue属性想支持别的类型也无能为力。

ConstantValue属性的结构如表6-23所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 | constant_value_index | 1                   |

- 从数据结构中可以看出ConstantValue属性是一个定长属性，它的attribute_length数据项值必须固定为2。
- constantvalue_index数据项代表了常量池中一个字面量常量的引用，
- 根据字段类型的不同，字面量可以是CONSTANT_Long_info、CONSTANT_Float_info、CONSTANT_Double_info、CONSTANT_Integer_info和CONSTANT_String_info常量中的一种。

#### InnerClasses属性

- InnerClasses属性用于记录**内部类与宿主类之间的关联**。
- 如果一个类中定义了内部类，那编译器将会为它以及它所包含的内部类生成InnerClasses属性。
- InnerClasses属性的结构如表6-24所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 | number_of_classes | 1                   |
| loca_classes_info |      inner_classes                     |number_of_classes |

数据项number_of_classes代表需要记录多少个内部类信息，

每一个内部类的信息都由一个inner_classes_info表进行描述。

inner_classes_info表的结构如表6-25所示。
| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 |inner_class_of_index     | 1                   |
| u4                 | outer_class_of_index          | 1                   |
| u2                 | inner_name_index | 1                   |
| loca_classes_info |      inner_classes_access_flag                     |1 |

- inner_class_info_index和outer_class_info_index都是指向常量池中CONSTANT_Class_info型常量的索引，分别代表了内部类和宿主类的符号引用。
- inner_name_index是指向常量池中CONSTANT_Utf8_info型常量的索引，代表这个内部类的名称，如果是匿名内部类，这项值为0。
- inner_class_access_flags是内部类的访问标志，类似于类的access_flags，

#### Deprecated及Synthetic属性

- Deprecated和Synthetic两个属性都属于标志类型的布尔属性，只存在有和没有的区别，没有属性值的概念。
- Deprecated属性用于表示某个类、字段或者方法，已经被程序作者定为不再推荐使用，它可以通过代码中使用“@deprecated”注解进行设置。
- Synthetic属性代表此字段或者方法并不是由Java源码直接产生的，而是由编译器自行添加的，
- 在JDK 5之后，标识一个类、字段或者方法是编译器自动产生的，也可以设置它们访问标志中的ACC_SYNTHETIC标志位。
- 编译器通过生成一些在源代码中不存在的Synthetic方法、字段甚至是整个类的方式，实现了越权访问（越过private修饰器）或其他绕开了语言限制的功能，这可以算是一种早期优化的技巧，其中最典型的例子就是枚举类中自动生成的枚举元素数组和嵌套类的桥接方法（Bridge Method）。
- 所有由不属于用户代码产生的类、方法及字段都应当至少设置Synthetic属性或者ACC_SYNTHETIC标志位中的一项，
- 唯一的例外是实例构造器“<init>()”方法和类构造器“<clinit>()”方法。

#### StackMapTable属性

- StackMapTable属性在JDK 6增加到Class文件规范之中，它是一个相当复杂的变长属性，位于Code属性的属性表中。

- 这个属性会在虚拟机类加载的字节码验证阶段被新类型检查验证器（TypeChecker）使用，目的在于代替以前比较消耗性能的基于数据流分析的类型推导验证器。

- StackMapTable属性中包含零至多个栈映射帧（Stack Map Frame），每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示执行到该字节码时局部变量表和操作数栈的验证类型。
- 类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。
- StackMapTable属性的结构如表6-28所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 | number_of_entries| 1                   |
| stack_map_frame |      stack_map_frame_entries                     |number_of_entries |

- 在Java SE 7版之后的《Java虚拟机规范》中，明确规定对于版本号大于或等于50.0的Class文件，如果方法的Code属性中没有附带StackMapTable属性，那就意味着它带有一个隐式的StackMap属性，这个StackMap属性的作用等同于number_of_entries值为0的StackMapTable属性。
- 一个方法的Code属性最多只能有一个StackMapTable属性，否则将抛出ClassFormatError异常。



#### Signature属性

- Signature属性在JDK 5增加到Class文件规范之中，它是一个可选的定长属性，可以出现于类、字段表和方法表结构的属性表中。
- 在JDK 5里面大幅增强了Java语言的语法，在此之后，任何类、接口、初始化方法或成员的泛型签名如果包含了类型变量（Type Variable）或参数化类型（ParameterizedType），则Signature属性会为它记录泛型签名信息。
- 之所以要专门使用这样一个属性去记录泛型类型，是因为Java语言的泛型采用的是擦除法实现的伪泛型，字节码（Code属性）中所有的泛型信息编译（类型变量、参数化类型）在编译之后都通通被擦除掉。
- 使用擦除法的好处是实现简单（主要修改Javac编译器，虚拟机内部只做了很少的改动）、非常容易实现Backport，运行期也能够节省一些类型所占的内存空间。
- 但坏处是运行期就无法像C#等有真泛型支持的语言那样，将泛型类型与用户定义的普通类型同等对待，例如运行期做反射时无法获得泛型信息。Signature属性就是为了弥补这个缺陷而增设的，
- 现在Java的反射API能够获取的泛型类型，最终的数据来源也是这个属性。
  Signature属性的结构如表6-29所示

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 |signature_index | 1                   |

- 其中signature_index项的值必须是一个对常量池的有效索引。常量池在该索引处的项必须是CONSTANT_Utf8_info结构，表示类签名或方法类型签名或字段类型签名。
- 如果当前的Signature属性是类文件的属性，则这个结构表示类签名，
- 如果当前的Signature属性是方法表的属性，则这个结构表示方法类型签名，
- 如果当前Signature属性是字段表的属性，则这个结构表示字段类型签名。

#### MethodParameters属性

- MethodParameters是在JDK 8时新加入到Class文件格式中的，它是一个用在方法表中的变长属性。
- MethodParameters的作用是记录方法的各个形参名称和信息。

##### 背景：

- 最初，基于存储空间的考虑，Class文件默认是不储存方法参数名称的，因为给参数起什么名字对计算机执行程序来说是没有任何区别的，所以只要在源码中妥当命名就可以了。
- 随着Java的流行，这点确实为程序的传播和二次复用带来了诸多不便，由于Class文件中没有参数的名称，如果只有单独的程序包而不附加JavaDoc的话，在IDE中编辑使用包里面的方法时是无法获得方法调用的智能提示的，这就阻碍了JAR包的传播。
- 后来，“-g：var”就成为了Javac以及许多IDE编译Class时采用的默认值，这样会将方法参数的名称生成到LocalVariableTable属性之中。
- 不过此时问题仍然没有全部解决，LocalVariableTable属性是Code属性的子属性——没有方法体存在，自然就不会有局部变量表，
- 但是对于其他情况，譬如抽象方法和接口方法，是理所当然地可以不存在方法体的，对于方法签名来说，还是没有找到一个统一完整的保留方法参数名称的地方。
- 所以JDK 8中新增的这个属性，使得编译器可以（编译时加上-parameters参数）将方法名称也写进Class文件中，而且MethodParameters是方法表的属性，与Code属性平级的，可以运行时通过反射API获取

MethodParameters的结构如表6-32所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u1                 | parameters_count| 1                   |
| parameter |      parameters                    |parameters_count |

其中，引用到的parameter结构如表6-33所示。
表6-33　parameter属性结构

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 |name_index       | 1                   |
| u2                 |access_flags | 1                   |



#### 模块化相关属性

- JDK 9的一个重量级功能是Java的模块化功能，
- 因为模块描述文件（module-info.java）最终是要编译成一个独立的Class文件来存储的，
- 所以，Class文件格式也扩展了***Module、ModulePackages和ModuleMainClas***s三个属性用于支持Java模块化相关功能

##### Module:

Module属性是一个非常复杂的变长属性，除了表示该模块的名称、版本、标志信息以外，还存储了这个模块***requires、exports、opens、uses***和***provides***定义的全部内容

Module属性是一个非常复杂的变长属性，除了表示该模块的名称、版本、标志信息以外，还存储
了这个模块requires、exports、opens、uses和provides定义的全部内容，其结构如表6-34所示。

表6-34　Module属性结构

![](/uploads/jvm/22-jvm-Module.png)

***TODO： 暂时用不到，就先不了解了***

##### ModulePackages

ModulePackages是另一个用于支持Java模块化的变长属性，它用于描述该模块中所有的包，不论是
不是被export或者open的。

##### ModuleMainClass

ModuleMainClass属性是一个定长属性，用于确定该模块的主类（Main Class），其结构



#### 运行时注解相关属性

##### 背景：

- 早在JDK 5时期，Java语言的语法进行了多项增强，其中之一是提供了对注解（Annotation）的支持。
- 为了存储源码中注解信息，Class文件同步增加了 ***RuntimeVisibleAnnotations***  、***RuntimeInvisibleAnnotations***、***RuntimeVisibleParameterAnnotations*** 和 ***RuntimeInvisibleParameterAnnotations***四个属性。
- 到了JDK 8时期，进一步加强了Java语言的注解使用范围，又新增类型注解（JSR 308），所以Class文件中也同步增加了 ***RuntimeVisibleTypeAnnotations*** 和
  ***RuntimeInvisibleTypeAnnotations*** 两个属性。
- 由于这六个属性不论结构还是功能都比较雷同，因此我们把它们合并到一起，以***RuntimeVisibleAnnotations***为代表进行介绍。
- **RuntimeVisibleAnnotations**是一个变长属性，它记录了类、字段或方法的声明上记录运行时可见注解，
- 当我们使用反射API来获取类、字段或方法上的注解时，返回值就是通过这个属性来取到的。
- 
- RuntimeVisibleAnnotations属性的结构如表6-38所示。

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | attribute_name_index       | 1                   |
| u4                 | attribute_length           | 1                   |
| u2                 | num_annotations| 1                   |
| annotation|      annotations                 |num_annotations |

num_annotations是annotations数组的计数器，annotations中每个元素都代表了一个运行时可见的注解，注解在Class文件中以annotation结构来存储，

具体如表6-39所示
表6-39　annotation属性结构

| 类型               | 名称                       | 数量                |
| ------------------ | -------------------------- | ------------------- |
| u2                 | type_index       | 1                   |
| u2                 |num_element_value_pairs           | 1                   |
| u2                 |element_value_pairs | num_element_value_pair                   |
