---
title: java高并发详解-10-JVM类加载器
copyright: true
comments: true
related_posts: true
date: 2021-03-01 22:46:10
tags: JVM类加载器
categories: 高并发详解
---

# JVM类加载器
- 类加载器就是负责类的加载的职责，对于任意一个class，都需要由加载它的类加载器和这个类本身确立其在JVM中的唯一性，这也就是 运行时包
- 任何一个对象的class 在JVM中只存在唯一的一份，比如 String.class 、Object.class 在堆内存以及方法区中肯定是唯一的 。
- 但是绝不可以理解为我们自定义的类 在JVM中同样也是这样。

## JVM内置三大类加载器

- JVM为我们提供了 三大内置的类加载器，不同的类加载器负责 将不同的类加载到JVM 内存之中，并且他们之间严格遵守着父委托的机制 

![](/uploads/java-concurrency-master/CLassLoader-father.png)



## 根类加载器介绍

根加载器又称为 Bootstrap类加载器，该类加载器是最为顶层的加载器，其没有任何父加载器，它是由 根加载器 所加载的，可以通过 -Xbootclasspath 来指定根加载器 的路径，也 可以通过系统属性来得知当前JVM 的根加载器都加载了哪些资源。

```java
public class BootStrapClassLoader
{
    public static void main(String[] args)
    {
        System.out.println("Bootstrap:" + String.class.getClassLoader());
        System.out.println(System.getProperty("sun.boot.class.path"));
    }
}
结果:
Bootstrap:null
D:\Java\Java8\jdk1.8.0\jre\lib\resources.jar;D:\Java\Java8\jdk1.8.0\jre\lib\rt.jar;D:\Java\Java8\jdk1.8.0\jre\lib\sunrsasign.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jsse.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jce.jar;D:\Java\Java8\jdk1.8.0\jre\lib\charsets.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jfr.jar;D:\Java\Java8\jdk1.8.0\jre\classes

```

## 扩展类加载器介绍

扩展类加载器的父加载器是根加载器，它主要用于加载 JAVA_HOME下的 jre\lib\ext 子目录里面的类库。扩展类加载器是由纯java语言实现的，它是 java.lang.URLClassLoader的子类，它的完整类名 是 sun.misc.Launcher$ExtClassLoader。扩展类加载器所加载的类库 可以通过系统属性 java.ext.dirs获得

```java
public class ExtClassLoader {
    public static void main(String[] args)
            throws ClassNotFoundException {
//        Class<?> helloClass = Class.forName("Hello");
//        System.out.println(helloClass.getClassLoader());
        System.out.println(System.getProperty("java.ext.dirs"));
    }
}

运行结果： D:\Java\Java8\jdk1.8.0\jre\lib\ext;C:\Windows\Sun\Java\lib\ext
```

## 系统类加载器介绍

系统类加载器是一种常见的类加载器，其负责加载 classpath下的类库资源。我们在进行项目开发的时候引入的第三方jar包，**系统类加载器的父类加载器 是扩展类加载器，同时它也是自定义类加载器的默认父加载器，**  系统类加载器的加载路径一般通过 -classpath 或者 -cp指定，同样的也可以通过系统属性 java.class.path进行获取，

```java
public class ApplicationClassLoader {
    public static void main(String[] args) {
        System.out.println(System.getProperty("java.class.path"));
        System.out.println(ApplicationClassLoader.class.getClassLoader());
    }
}

结果： 
    D:\Java\Java8\jdk1.8.0\jre\lib\charsets.jar;D:\Java\Java8\jdk1.8.0\jre\lib\deploy.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\access-bridge-64.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\cldrdata.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\dnsns.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\jaccess.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\jfxrt.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\localedata.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\nashorn.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\sunec.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\sunjce_provider.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\sunmscapi.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\sunpkcs11.jar;D:\Java\Java8\jdk1.8.0\jre\lib\ext\zipfs.jar;D:\Java\Java8\jdk1.8.0\jre\lib\javaws.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jce.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jfr.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jfxswt.jar;D:\Java\Java8\jdk1.8.0\jre\lib\jsse.jar;D:\Java\Java8\jdk1.8.0\jre\lib\management-agent.jar;D:\Java\Java8\jdk1.8.0\jre\lib\plugin.jar;D:\Java\Java8\jdk1.8.0\jre\lib\resources.jar;D:\Java\Java8\jdk1.8.0\jre\lib\rt.jar;H:\pdf\java-concurrency-master\java-concurrency-master\book\target\classes;C:\Users\yinshi\.m2\repository\mysql\mysql-connector-java\6.0.6\mysql-connector-java-6.0.6.jar;D:\Program Files\JetBrains\IntelliJ IDEA 2020.1.3\lib\idea_rt.jar

        sun.misc.Launcher$AppClassLoader@14dad5dc
```

# 自定义类加载器

用程序实现自定义的类加载器，所有的自定义类加载都是ClassLoader的直接子类或者间接子类，java.lang.ClassLoader是一个抽象类，它里面并没有抽象方法，但是又findClass 方法，务必实现该方法，否则将会抛出 Class找不到的异常

```java
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
```

```java
/**
 * 自定义类加载器必须是ClassLoader 的直接或者间接子类
 */
public class MyClassLoader extends ClassLoader {
    // 定义默认的class 存放路径
    private final static Path DEFAULT_CLASS_DIR = Paths.get("H:\\pdf\\java-concurrency-master\\java-concurrency-master\\book\\target\\classes\\com\\wangwenjun\\concurrent\\chapter10"
            , "classloader1");

    private final Path classDir;

    // 使用默认的class路径
    public MyClassLoader() {
        super();
        this.classDir = DEFAULT_CLASS_DIR;
    }

    //允许传入指定的class路径
    public MyClassLoader(String classDir) {
        super();
        this.classDir = Paths.get(classDir);
    }

    //指定class路径的同时，指定父类加载器
    public MyClassLoader(String classDir, ClassLoader parent) {
        super(parent);
        this.classDir = Paths.get(classDir);
    }

    //重写父类的findClass 方法，这个是至关重要的step
    @Override
    protected Class<?> findClass(String name)
            throws ClassNotFoundException {
        // 读取class 的二进制数据
        byte[] classBytes = this.readClassBytes(name);
        //如果数据为null，或者没有读到任何信息，则抛出 ClassNotFoundException 异常
        if (null == classBytes || classBytes.length == 0) {
            throw new ClassNotFoundException("Can not load the class " + name);
        }
        //调用defineClass 方法定义class（）
        //name 定义类的名字
        // classBytes class文件的二进制字节数组
        // 字节数组的偏移量
        // 从偏移量开始读取多长的byte数据
        // 问题： 第一个 阶段的加载主要是获取class的字节流信息，那么我们将整个字节流信息交给defineClass不就行 了吗，为什么还要指定偏移量和读取长度呢：
        // 因为class字节数组不一定是从一个class文件中获得的，有可能是来自网络的，也有可能是来自网络或者其他途径。
        //由此可见一个字节数组中很有可能存储多个class的字节信息
        return this.defineClass(name, classBytes, 0, classBytes.length);
    }

    /**
     * 将class文件读入内存
     *
     * @param name
     * @return
     * @throws ClassNotFoundException
     */
    private byte[] readClassBytes(String name)
            throws ClassNotFoundException {
        // 将包名分隔符 转换为分拣路径分隔符
        String classPath = name.replace(".", "/");
        Path classFullPath = classDir.resolve(Paths.get(classPath + ".class"));
        if (!classFullPath.toFile().exists())
            throw new ClassNotFoundException("The class " + name + " not found.");
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            Files.copy(classFullPath, baos);
            return baos.toByteArray();
        } catch (IOException e) {
            throw new ClassNotFoundException("load the class " + name + " occur error.", e);
        }
    }

    @Override
    public String toString() {
        return "My ClassLoader";
    }
}
```

- 第一个构造函数使用默认的文件路径
- 第二个构造函数允许外部指定一个特定的磁盘目录
- 第三个构造函数除了可以指定磁盘目录外，还可以指定该类加载的父加载器



全路径格式有以下几种情况：

- java.lang.String:     包名.类名
- javax.swing.JSpinner $DefaultEditor: 包名.类名 $内部类
- java.security.KeyStor$Builder$FileBuilder$1 :包名.类名 $内部类 $内部类$匿名内部类
- java.net.URLClassLoader$3$1: 包名.类名 $内部类$匿名内部类



强调defineClass 方法，该方法的完整方法描述是  defineClass(String name, byte[] b, int off, int len) 

- 其中第一个是要定义类的名字，一般与findClass 方法中的类名保持一致即可
- 第二个是 class文件的二进制字节数组，
- 第三个是字节数组的偏移量
- 第4个是从偏移量开始读取多长的byte数据





## 双亲委托机制详细介绍

- 当一个类加载器被调用了loadClass之后，它并不会直接将其加载，而是先交给当前类加载器的 父加载器尝试加载直到最顶层的父加载器，然后一次向下进行加载。

![](/uploads/java-concurrency-master/classLoaderOfFather.png)

在解析loadClass 源码之前，思考一个问题，由于担心HelloWorld.class 被系统类加载器加载，所以删除了HelloWorld 的相关文件，那么有什么办法可以不用删除又可以使用 MyClassLoader对HelloWorld 进行加载的吗？



```java
 public Class<?> loadClass(String name) throws ClassNotFoundException {
     return loadClass(name, false);
}

 protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

上面的代码片段是 java.lang.ClassLoader 的loadClass（name）和 loadClass（name，resolve）方法，由于loadClass （name）调用的是 loadClass（name，false），因此我们重点解释  loadClass（name，false）即可

- 从当前类加载器的已加载类缓存中根据类的全路径名查询是否存在该类， 如果存在则直接返回 
- 如果当前类存在父类加载器，则 调用父类加载器的loadClass （name,false） 方法对其 进行加载
- 如果当前类加载不存在父类加载器，则直接调用根类加载器 对该类进行加载
- 如果 当前类的所有父类加载器都没有成功加载class ，则尝试 调用当前类加载器的 findClass 方法对其进行加载，该方法就是我们自定义加载器需要重写的方法
- 最后如果类被成功加载，则做一些性能数据的统计 
- 由于loadClass 指定了revolve 为false，所以不会 进行连接阶段的继续执行 ，这也就解释了  为什么通过类加载器 加载类并不会导致类的初始化



### 问题

如何在不删除HelloWorld.class 文件的情况下 使用MyClassLoader 而不是系统类加载器进行HelloWorld 的加载，有如下两种方法可以做到。

1. 第一种方式 是绕过系统类加载器，直接将扩展类加载器 作为MyClassLoader 的父加载器，示例 代码如下： 

```java
ClassLoader extClassloader = MyClassLoaderTest.class.getClassLoader().getParent();
MyClassLoader classLoader = new MyClassLoader("G:\\classLoader1",extClassloader);
Class<?> aclass=classLoader.loadClass("com.wangwenjun.concurrent.chapter10.HelloWorld");
System.out.println(aClass);
System.out.println(aClass.getClassLoader());

```

- 首先我们通过MyClassLoaderTest.class 获取系统类加载器，然后再获取系统类加载器的父类加载器 扩展类加载器，使其成为MyClassLoader的父类加载器，这样一来，根加载器和扩展类加载器都无法对 G:\\ classloader 类文件进行加载，自然而然就交给了MyClassLoader 对 HelloWorld 进行加载了，这种方式充分利用了类加载器 父类委托机制的特性

2.  第二种方式实在构造 MyClassLoader 的时候指定其父类加载器为null ，示例代码如下：

```java
MyClassLoader classLoader=new MyClassLoader("G:\\classloader1",null);
Class<?> aclass=classLoader.loadClass("com.wangwenjun.concurrent.chapter10.HelloWorld");
System.out.println(aClass);
System.out.println(aClass.getClassLoader());

根据对 loadClass方法的源码分析，当前类在没有父类加载器的情况下，会直接使用根加载器对该类进行加载 ，很显然，HelloWorld 在根加载器的加载路径下 是无法找到的，那么它 自然而然地就交给当前类加载器进行加载了

```



## 破坏双亲委托机制



- 我们发现类加载器的父委托机制的逻辑 主要是由loadClass来控制的，有些时候我们需要打破这种双亲委托的机制，比如 HelloWorld 这个类就是不希望通过系统类加载器对其进行加载。
- JDk 提供的双亲委托机制并非一个强制性的模型，程序开发人员是可以对其进行 灵活发挥破坏这种委托机制的 

比如： 如果我们想要在程序运行时进行某个模块功能的升级，甚至是 在不停止服务的前提下增加新的功能，这就是我们常说的热部署。

- 热部署首先要卸载掉加载该模块所有Class的类加载器，卸载类加载器会导致所有类的卸载，
- 很显然我们无法对JVM 三大内置加载器进行卸载，我们只有通过控制 自定义类加载器才能做到这一点 

我们可以通过破坏父委托机制的方式 来实现对HelloWorld类的加载，而不需要在工程中删除该文件

```java
public class BrokerDelegateClassLoader extends ClassLoader {

    private final static Path DEFAULT_CLASS_DIR = Paths.get("G:", "classloader1");

    private final Path classDir;

    public BrokerDelegateClassLoader() {
        super();
        this.classDir = DEFAULT_CLASS_DIR;
    }

    public BrokerDelegateClassLoader(String classDir) {
        super();
        this.classDir = Paths.get(classDir);
    }

    public BrokerDelegateClassLoader(String classDir, ClassLoader parent) {
        super(parent);
        this.classDir = Paths.get(classDir);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = this.readClassBytes(name);
        if (null == classBytes || classBytes.length == 0) {
            throw new ClassNotFoundException("Can not load the class " + name);
        }

        return this.defineClass(name, classBytes, 0, classBytes.length);
    }

    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // 1. 根据类的全路径名称进行加锁，确保每一个类在多线程 的情况下制备加载一次 
        synchronized (getClassLoadingLock(name)) {
            //2. 到已加载类的缓存中查看该类是否已经被加载，如果已加载则直接返回
            Class<?> klass = findLoadedClass(name);
            //3.
            if (klass == null) {
                //4. 假如缓存中没有被加载的类，则需要对其进行首次加载，如果类的全路径以java和javax开头，则直接委托给 系统类加载器对其进行加载 
                if (name.startsWith("java.") || name.startsWith("javax")) {
                    try {
                        klass = getSystemClassLoader().loadClass(name);
                    } catch (Exception e) {
                        //ignore
                    }
                } else {
                    // 5. 如果类不是以java 和javax 开头，则尝试用我们自定义的类加载进行加载 
                    try {
                        klass = this.findClass(name);
                    } catch (ClassNotFoundException e) {
                        //ignore
                    }
                    // 6. 如果 自定义类加载仍旧 没有完成对类的加载，则委托 给其父类加载器进行加载或者系统类加载器进行加载 
                    if (klass == null) {
                        if (getParent() != null) {
                            klass = getParent().loadClass(name);
                        } else {
                            klass = getSystemClassLoader().loadClass(name);
                        }
                    }
                }
            }
            // 7. 经过诺干次的尝试后，如果还是无法对类进行加载，则抛出无法找到类的异常 
            if (null == klass) {
                throw new ClassNotFoundException("The class " + name + " not found.");
            }
            if (resolve) {
                resolveClass(klass);
            }
            return klass;
        }
    }

    private byte[] readClassBytes(String name) throws ClassNotFoundException {
        String classPath = name.replace(".", "/");
        Path classFullPath = classDir.resolve(Paths.get(classPath + ".class"));
        if (!classFullPath.toFile().exists())
            throw new ClassNotFoundException("The class " + name + " not found.");
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            Files.copy(classFullPath, baos);
            return baos.toByteArray();
        } catch (IOException e) {
            throw new ClassNotFoundException("load the class " + name + " occur error.", e);
        }
    }

    @Override
    public String toString() {
        return "Broker Delegate ClassLoader";
    }
}
```



## 类加载器命名空间、运行时包、类的卸载等

### 1.类的命名空间

每一个类加载器都有各自的命名空间，命名空间时由该加载器及其所有父加载器所构成的，因此在每一个类加载器中同一个class 都是独一无二的，

```java
 public static void main(String[] args) throws ClassNotFoundException {
        MyClassLoader classLoader1 = new MyClassLoader("G:\\classloader1", null);
        MyClassLoader classLoader2 = new MyClassLoader("G:\\classloader1", null);
        Class<?> aClass = classLoader1.loadClass("com.wangwenjun.concurrent.chapter10.Test");
        Class<?> bClass = classLoader2.loadClass("com.wangwenjun.concurrent.chapter10.Test");
        System.out.println(aClass.getClassLoader());
        System.out.println(bClass.getClassLoader());
        System.out.println(aClass.hashCode());
        System.out.println(bClass.hashCode());
        System.out.println(aClass == bClass);
    }
```

运行上面的代码，不论load多少次Test，都会发现他们始终时同一份class对象，这也完全符合我们在本书9.1节中的描述。 类被加载后的内存情况如图所示：

![](/uploads/java-concurrency-master/classAfterLoader.png)

- 但是，使用不同的类加载器，或者同一个类加载器的不同示例，去加载同一个class，则会在堆内存和方法区产生多个class对象 

（1） 不同类加载器加载同一个class，输出的结果显示aclass 和 bclass不是同一个class示例 

（2）相同类加载器加载同一个class,输出的结果显示aclass 和 bclass不是同一个class示例 



分析JDK 中关于ClassLoader 的相关源代码，具体如下：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            。。。
                
  protected final Class<?> findLoadedClass(String name) {
        if (!checkName(name))
            return null;
        return findLoadedClass0(name);
    }

    private native final Class<?> findLoadedClass0(String name);

```

在类加载器进行类加载的时候，首先会到  加载记录表也就是缓存中，查看该类是否已经被加载过了，如果已经被加载过了，就不会重复加载，否则就会认为其是 首次加载， 下图就是同一个class 被不同类加载器加载之后的内存i情况 。

![](/uploads/java-concurrency-master/differentClassLoaderLoadClass.png)

同一个class示例在同一个类加载器命名空间之下是唯一的。

### 2. 运行时包(TODO)