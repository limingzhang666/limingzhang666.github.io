---
title: 深入理解jvm-10-Java模块化系统
copyright: true
related_posts: true
date: 2021-01-18 23:00:29
tags: Java模块化系统
categories: jvm
---

# Java模块化系统(TODO DODO)



在JDK 9中引入的Java模块化系统（Java Platform Module System，JPMS）是对Java技术的一次重要升级，为了能够实现模块化的关键目标——可配置的封装隔离机制，Java虚拟机对类加载架构也做出了相应的变动调整，才使模块化系统得以顺利地运作。JDK 9的模块不仅仅像之前的JAR包那样只是简单地充当代码的容器，除了代码外，Java的模块定义还包含以下内容：

- 依赖其他模块的列表。
- 导出的包列表，即其他模块可以使用的列表。
- 开放的包列表，即其他模块可反射访问模块的列表。
- 使用的服务列表。
- 提供服务的实现列表



## 可配置的封装隔离机制

- 可配置的封装隔离机制首先要解决JDK 9之前基于**类路径（ClassPath）来查找依赖**的可靠性问题。此前，如果类路径中缺失了运行时依赖的类型，那就只能等程序运行到发生该类型的加载、链接时才会报出运行的异常
- 而在JDK 9以后，如果启用了**模块化进行封装**，模块就可以声明对其他模块的**显式依赖**，这样Java虚拟机就能够在**启动时验证**应用程序开发阶段设定好的依赖关系在运行期是否完备，如有缺失那就直接启动失败，从而避免了很大一部分[1]由于类型依赖而引发的运行时异常
- 可配置的封装隔离机制还**解决了原来类路径上跨JAR文件的public类型的可访问性问题**。JDK 9中的public类型不再意味着程序的所有地方的代码都可以随意访问到它们，模块提供了更**精细的可访问性控制**，必须明确声明其中哪一些public的类型可以被其他哪一些模块访问，这种**访问控制也主要是在类加载过程中**完成的



## 模块的兼容性

- 为了使可配置的封装隔离机制能够兼容传统的类路径查找机制，JDK 9提出了与“**类路径**”（ClassPath）相对应的“**模块路径**”（ModulePath）的概念
- 就是某个类库到底是模块还是传统的JAR包，只取决于**它存放在哪种路径上**。
- 只要是放在**类路径上的JAR文件**，无论其中是否包含模块化信息（是否包含了module-info.class文件），它都会被当作**传统的JAR包**来对待；
- 相应地，只要放在**模块路径上的JAR文件**，即使没有使用JMOD后缀，甚至说其中并不包含module-info.class文件，它也仍然会被当作一个**模块**来对待。



### 保障规则：

模块化系统将按照以下规则来保证使用传统类路径依赖的Java程序可以不经修改地直接运行在JDK 9及以后的Java版本上，即使这些版本的JDK已经使用模块来封装了Java SE的标准类库，模块化系统的这套规则也仍然保证了传统程序可以访问到所有标准类库模块中导出的包

#### JAR文件在类路径的访问规则

- JAR文件在类路径的访问规则：所有类路径下的JAR文件及其他资源文件，都被视为自动打包在一个匿名模块（Unnamed Module）里，这个匿名模块几乎是没有任何隔离的，它可以看到和使用类路径上所有的包、JDK系统模块中所有的导出包，以及模块路径上所有模块中导出的包。

#### 模块在模块路径的访问规则

- 模块在模块路径的访问规则：模块路径下的具名模块（Named Module）只能访问到它依赖定义中列明依赖的模块和包，匿名模块里所有的内容对具名模块来说都是不可见的，即具名模块看不见传统JAR包的内容。

#### JAR文件在模块路径的访问规则

- JAR文件在模块路径的访问规则：如果把一个传统的、不包含模块定义的JAR文件放置到模块路径中，它就会变成一个自动模块（Automatic Module）。尽管不包含module-info.class，但自动模块将默认依赖于整个模块路径中的所有模块，因此可以访问到所有模块导出的包，自动模块也默认导出自己所有的包。



以上3条规则保证了即使Java应用依然使用传统的类路径，升级到JDK 9对应用来说几乎（类加载器上的变动还是可能会导致少许可见的影响，将在下节介绍）不会有任何感觉，项目也不需要专门为了升级JDK版本而去把传统JAR包升级成模块。

除了向后兼容性外，随着JDK 9模块化系统的引入，更值得关注的是它本身面临的模块间的管理和兼容性问题：如果同一个模块发行了多个不同的版本，那只能由开发者在编译打包时人工选择好正确版本的模块来保证依赖的正确性。Java模块化系统目前不支持在模块定义中加入版本号来管理和约束依赖，本身也不支持多版本号的概念和版本选择功能。

我们不论是在Java命令、Java类库的API抑或是《Java虚拟机规范》定义的Class文件格式里都能轻易地找到证据，表明模块版本应是编译、加载、运行期间
都可以使用的。譬如输入“java--list-modules”，会得到明确带着版本号的模块列表：

```bash
java.base@12.0.1
java.compiler@12.0.1
java.datatransfer@12.0.1
java.desktop@12.0.1
java.instrument@12.0.1
java.logging@12.0.1
java.management@12.0.1
....
```

在JDK 9时加入Class文件格式的**Module属性**，里面有**module_version_index**这样的字段，用户可以在编译时使用“**javac--module-version**”来指定模块版本，在Java类库API中也存在java.lang.module.ModuleDescriptor.Version这样的接口可以在运行时获取到模块的版本号。这一切迹象都证明了**Java模块化系统对版本号的支持本可以不局限在编译期**。



## 模块化下的类加载器

为了保证兼容性，JDK 9并没有从根本上动摇从JDK 1.2以来运行了二十年之久的三层类加载器架构以及双亲委派模型。但是为了模块化系统的顺利施行，模块化下的类加载器仍然发生了一些应该被注意到变动，主要包括以下几个方面。

### 扩展类加载器被取代

- 首先，是扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代。
- 这其实是一个很顺理成章的变动，既然整个JDK都基于模块化进行构建（原来的rt.jar和tools.jar被拆分成数十个JMOD文件），其中的Java类库就已天然地满足了可扩展的需求，那自然无须再保留**<JAVA_HOME>\lib\ext**目录，此前使用这个目录或者java.ext.dirs系统变量来扩展JDK功能的机制已经没有继续存在的价值了，用来加载这部分类库的扩展类加载器也完成了它的历史使命。
- 类似地，在新版的JDK中也**取消了<JAVA_HOME>\jre**目录，因为随时可以组合构建出程序运行所需的JRE来，譬如假设我们只使用java.base模块中的类型，那么随时可以通过以下命令打包出一个“JRE”：

```bash
jlink -p $JAVA_HOME/jmods --add-modules java.base --output jre
```

### BuiltinClassLoade

- 其次，平台类加载器和应用程序类加载器都不再派生自java.net.URLClassLoader，如果有程序直接依赖了这种继承关系，或者依赖了URLClassLoader类的特定方法，那代码很可能会在JDK 9及更高版本的JDK中崩溃。
- 现在启动类加载器、平台类加载器、应用程序类加载器全都继承于jdk.internal.loader.BuiltinClassLoader，在BuiltinClassLoader中实现了新的模块化架构下类如何从模块中加载的逻辑，以及模块中资源可访问性的处理。

![](/uploads/jvm/09ClassLoader/01-ClassLoader_before.png)

![](source/uploads/jvm/09ClassLoader/02-ClassLoader_after.png)

图7-6中有“BootClassLoader”存在，启动类加载器现在是在Java虚拟机内部和Java类库共同协作实现的类加载器，尽管有了BootClassLoader这样的Java类，但为了与之前的代码保持兼容，所有在获取启动类加载器的场景（譬如Object.class.getClassLoader()）中仍然会返回null来代替，而不会得到BootClassLoader的实例。





### 类加载的委派关系发生了变动

![](source/uploads/jvm/09ClassLoader/03-ClassLoader_Parentnew.png)

JDK 9中虽然仍然维持着三层类加载器和双亲委派的架构，但类加载的委派关系也发生了变动。

当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责那个模块的加载器完成加载，也许这可以算是对双亲委派的第四次破坏。

#### 类加载器负责加载的模块

##### 启动类加载器负责加载的模块

```bash
java.base java.security.sasl
java.datatransfer java.xml
java.desktop jdk.httpserver
java.instrument jdk.internal.vm.ci
java.logging jdk.management
java.management jdk.management.agent
java.management.rmi jdk.naming.rmi
java.naming jdk.net
java.prefs jdk.sctp
java.rmi jdk.unsupported
```



##### 平台类加载器负责加载的模块

```bash
java.activation* 					jdk.accessibility
java.compiler* 						jdk.charsets
java.corba* 						jdk.crypto.cryptoki
java.scripting 						jdk.crypto.ec
java.se 							jdk.dynalink
java.se.ee 							jdk.incubator.httpclient
java.security.jgss					jdk.internal.vm.compiler*
java.smartcardio 					jdk.jsobject
java.sql 							jdk.localedata
java.sql.rowset						jdk.naming.dns
java.transaction* 					jdk.scripting.nashorn
java.xml.bind* 						jdk.security.auth
java.xml.crypto 					jdk.security.jgss
java.xml.ws* 						jdk.xml.dom
java.xml.ws.annotation* 			jdk.zipfs
```

##### 应用程序类加载器负责加载的模块

```bash
jdk.aot 						jdk.jdeps
jdk.attach 						jdk.jdi
jdk.compiler 					jdk.jdwp.agent
jdk.editpad 					jdk.jlink
jdk.hotspot.agent 				jdk.jshell
jdk.internal.ed					jdk.jstatd
jdk.internal.jvmstat 			jdk.pack
jdk.internal.le 				jdk.policytool
jdk.internal.opt 				jdk.rmic
jdk.jartool 					jdk.scripting.nashorn.shell
jdk.javadoc 					jdk.xml.bind*
jdk.jcmd 						jdk.xml.ws*
jdk.jconsole
```



