---
title: 深入理解jvm-13-类加载及执行子系统的案例与实战
copyright: true
related_posts: true
date: 2021-01-25 22:22:30
tags: 类加载及执行子系统的案例与实战
categories: jvm
---

# 类加载及执行子系统的案例与实战

在Class文件格式与执行引擎这部分里，用户的程序能直接参与的内容并不太多，Class文件以何种格式存储，类型何时加载、如何连接，以及虚拟机如何执行字节码指令等都是由虚拟机直接控制的行为，用户程序无法对其进行改变.

能通过程序进行操作的，主要是字节码生成与类加载器这两部分的功能，但仅仅在如何处理这两点上，就已经出现了许多值得欣赏和借鉴的思路，这些思路后来成为许多常用功能和程序实现的基础。

## 案例分析

### Tomcat：正统的类加载器架构

主流的Java Web服务器，如Tomcat、Jetty、WebLogic、WebSphere或其他笔者没有列举的服务器，都实现了自己定义的类加载器，而且一般还都不止一个。因为一个功能健全的Web服务器，都要解决如下的这些问题：

1. 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以实现**相互隔离**。这是最基本的需求，两个不同的应用程序可能会依赖同一个第三方类库的不同版本，不能要求每个类库在一个服务器中只能有一份，服务器应当能够**保证两个独立应用程序的类库可以互相独立使用。**
2. 部署在同一个服务器上的两个Web应用程序所使用的Java类库可以互相共享。这个需求与前面一点正好相反，但是也很常见，例如用户可能有10个使用Spring组织的应用程序部署在同一台服务器上，如果把10份Spring分别存放在各个应用程序的隔离目录中，将会是很大的资源浪费——这主要倒不是浪费磁盘空间的问题，而是指类库在使用时都要被加载到服务器内存，如果类库不能共享，虚拟机的方法区就会很容易出现过度膨胀的风险。
3. 服务器需要尽可能地保证自身的安全不受部署的Web应用程序影响。目前，有许多主流的JavaWeb服务器自身也是使用Java语言来实现的。因此服务器本身也有类库依赖的问题，一般来说，基于安全考虑，服务器所使用的类库应该与应用程序的类库互相独立。
4. 支持JSP应用的Web服务器，十有八九都需要支持HotSwap功能。我们知道JSP文件最终要被编译成Java的Class文件才能被虚拟机执行，但JSP文件由于其纯文本存储的特性，被运行时修改的概率远大于第三方类库或程序自己的Class文件。而且ASP、PHP和JSP这些网页应用也把修改后无须重启作为一个很大的“优势”来看待，因此“主流”的Web服务器都会支持JSP生成类的热替换，当然也有“非主流”的，如运行在生产模式（Production Mode）下的WebLogic服务器默认就不会处理JSP文件的变化。

由于存在上述问题，在部署Web应用时，单独的一个ClassPath就不能满足需求了，所以各种Web服务器都不约而同地**提供了好几个有着不同含义的ClassPath路径**供用户存放第三方类库，这些路径一般会以“lib”或“classes”命名。被放置到不同路径中的类库，具备不同的访问范围和服务对象，通常每一个目录都会有一个相应的自定义类加载器去加载放置在里面的Java类库。现在就以Tomcat服务器[1]为例，与读者一同分析Tomcat具体是如何规划用户类库结构和类加载器的。

在Tomcat目录结构中，可以设置3组目录（/**common**/*、/**server**/*和/**shared**/*，但默认不一定是开放的，可能只有/lib/*目录存在）用于存放Java类库，另外还应该加上Web应用程序自身的“/**WEBINF**/*”目录，一共4组。把Java类库放置在这4组目录中，每一组都有独立的含义，分别是：

- 放置在/common目录中。类库可被Tomcat和所有的Web应用程序共同使用。
- 放置在/server目录中。类库可被Tomcat使用，对所有的Web应用程序都不可见。
- 放置在/shared目录中。类库可被所有的Web应用程序共同使用，但对Tomcat自己不可见。
- 放置在/WebApp/WEB-INF目录中。类库仅仅可以被该Web应用程序使用，对Tomcat和其他Web应用程序都不可见。

为了支持这套目录结构，并对目录里面的类库进行加载和隔离，**Tomcat自定义了多个类加载器**，这些类加载器按照经典的双亲委派模型来实现，其关系如图9-1所示。

![](/uploads/jvm/23-tomcatClassLoader.png)

- 灰色背景的3个类加载器是JDK（以JDK 9之前经典的三层类加载器为例）默认提供的类加载器，这3个加载器的作用在第7章中已经介绍过了。
- 而Common类加载器、Catalina类加载器（也称为Server类加载器）、Shared类加载器和Webapp类加载器则是Tomcat自己定义的类加载器，它们分别加
  载/common/*、/server/*、/shared/*和/WebApp/WEB-INF/*中的Java类库。
- 其中WebApp类加载器和JSP类加载器通常还会存在多个实例，每一个Web应用程序对应一个WebApp类加载器，每一个JSP文件对应一个JasperLoader类加载器。



从图9-1的委派关系中可以看出，

- Common类加载器能加载的类都可以被Catalina类加载器和Shared类加载器使用，
- 而Catalina类加载器和Shared类加载器自己能加载的类则与对方相互隔离。
- WebApp类加载器可以使用Shared类加载器加载到的类，但各个WebApp类加载器实例之间相互隔离。
- 而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个Class文件，它存在的目的就是为了被丢弃：当服务器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的JSP类加载器来实现JSP文件的HotSwap功能。



本例中的类加载结构在**Tomcat 6以前**是它默认的类加载器结构，在Tomcat 6及之后的版本简化了默认的目录结构，只有指定了**tomcat/conf/catalina.properties**配置文件的**server.loader和share.loader**项后才会真正建立Catalina类加载器和Shared类加载器的实例，否则会用到这两个类加载器的地方都会用Common类加载器的实例代替，

而默认的配置文件中并没有设置这两个loader项，所以Tomcat 6之后也顺理成章地把/common、/server和/shared这3个目录默认合并到一起变成1个/lib目录，这个目录里的类库相当于以前/common目录中类库的作用，是Tomcat的开发团队为了简化大多数的部署场景所做的一项易用性改进。

如果默认设置不能满足需要，用户可以通过修改配置文件指定server.loader和share.loader的方式重新启用原来完整的加载器架构。

#### Tomcat类加载过程

（1）在Tomcat启动时，会创建一系列的类加载器，在其主类Bootstrap的初始化过程中，会先初始化classloader，然后将其绑定到Thread中。 

（2）其中initClassLoaders方法，会根据catalina.properties的配置，创建相应的classloader。由于默认只配置了common.loader属性，所以其中只会创建一个出来

（3）然后，当一个应用启动的时候，会为其创建对应的WebappClassLoader。此时会将commonClassLoader设置为其parent

（4）在加载时，与java双亲委托机制不同，默认是子优先，也就是先用子加载器加载但是保证Java的基础类不允许其重新加载，以及servlet-api也不允许重新加载。



#### 问题：

那么笔者不妨再提一个问题让各位读者思考一下：前面曾经提到过一个场景，如果有10个Web应用程序都是用Spring来进行组织和管理的话，可以把Spring
放到Common或Shared目录下让这些程序共享。Spring要对用户程序的类进行管理，自然要能访问到用户程序的类，而用户的程序显然是放在/WebApp/WEB-INF目录中的。那么被Common类加载器或Shared类加载器加载的Spring如何访问并不在其加载范围内的用户程序呢？

#### 答案：

**当然就是前片所说的使用线程上下文类加载器，当我们在spring的配置文件中通过类来创建对象，spring都是使用线程上下文类加载器来加载了，默认设置为WebAppClassLoader，所以当在web应用程序内，spring通过线程上下文类加载器使用WebAppClassLoader来加载bean**

答案明显是使用线程上下文类加载器来实现的啊！仔细看源码你会发现，spring加载类所用的classloader都是通过Thread.currentThread().getContextClassLoader()来获取的，而当线程创建时会默认 setContextClassLoader(AppClassLoader)，即spring中始终可以获取到这个AppClassLoader(在tomcat里就是WebAppClassLoader)子类加载器来加载bean，以后任何一个线程都可以通过getContextClassLoader()获取到WebAppClassLoader来getbean了





### 　OSGi：灵活的类加载器架构

曾经在Java程序社区中流传着这么一个观点：“学习Java EE规范，推荐去看JBoss源码；学习类加载器的知识，就推荐去看OSGi源码。



