---
layout: post
title:  "JVM-ClassLoader类加载器"
date:   2019-08-18 08:38:00
categories: Java 
tags: Java JVM
---

* content
{:toc}

ClassLoad类装载器在Java中有着非常重要的作用，主要工作在Class装载的加载阶段，主要作用是从系统外部获得Class二进制数据流






## ClassLoader类：

`所有的Class都是由ClassLoader进行加载的，ClassLoader负责通过各种方式将Class二进制数据流读入系统，然后交给Java虚拟机进行连接，初始化等操作`。ClassLoader在整个装载阶段，只能影响到类的加载，而无法通过ClassLoader去改变类的连接和初始化行为

ClassLoader类是一个抽象类，它提供了一些重要的接口用于自定义Class的加载流程和加载方式。主要方法如下：


- loadClass(String name) 

给定一个类名，加载一个类，返回代表这个类的Class实例，如果找不到，则抛出ClassNotFoundException异常

- defineClass(String name, byte[] b, int off, int len)：受保护方法。只能在自定义ClassLoader子类中使用

根据给定的字节码流定一个类

- findClass(String name)：受保护方法，重载ClassLoader时，重要的扩展点

查找一个类，这个方法会在loadClass()时被调用，用于自定义查找类的逻辑。如果不需要修改类加载默认机制，只是想改变类加载形式，就可以重载该方法

- findLoaderClass(String name)

final方法，无法被修改。寻找已经加载的类

在ClassLoader的结构中，还有一个重要的字段parent，是一个ClassLoader的实例，这个属性所表示的ClassLoader称为这个ClassLoader的双亲。在类的加载过程中，ClassLoader可能会将某些请求交予自己的双亲处理


## ClassLoader分类

在标准的Java应用中，Java虚拟机会创建3个ClassLoader为整个应用程序服务，分别为：`BootStrap ClassLoader(启动类加载器)、Extension ClassLoader(扩展类加载器)、APP ClassLoader(应用类加载器，也称系统类加载器)`。此外，每个应用程序还可以拥有自定义的ClassLoader，扩展Java虚拟机获取Class数据的能力。如下ClassLoader层次结构图：
![](\img\post.img\jvm\classloader-level.png)


ClassLoader的层次自顶向下为启动类加载器、扩展类加载器、应用类加载器和自定义类加载器，其中应用类加载器的双亲为扩展类加载器，扩展类加载器的双亲为启动类加载器。`当系统需要使用一个类时，在判断类是否已经被加载时，会先从当前底层类加载器进行判断，当系统需要加载一个类时，会从顶层类考试加载，一次向下尝试，直到成功`。在这些ClassLoader中，启动类加载器最为特别，由C代码实现，并且在Java中没有对象与之对应。`系统的核心类就是由启动类加载器进行的`，它也是虚拟机的核心组件。扩展类加载器和应用类加载器都有对应的Java对象可供使用。如下获取类的所有加载器：

```java
public class PrintClassLoaderTree {

    public static void main(String[] args) {
        ClassLoader classLoader = PrintClassLoaderTree.class.getClassLoader();

        while (null != classLoader) {
            System.out.println(classLoader);
            classLoader = classLoader.getParent();//获取父类加载器
        }
    }
}

输出：
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$ExtClassLoader@404b9385

```

PrintClassLoaderTree用户类加载于AppClassLoader(应用类加载器)中，而AppClassLoader的双亲为ExtClassLoader(扩展类加载器)，ExtClassLoader无法再获取启动类加载器。因此`任何加载在启动类加载器中的类是无法获得其ClassLoader实例的`。如:

```java
System.out.println(String.class.getClassLoader()); //由于String属于Java核心类，因此会被启动类加载器加载，因此打印null
//null
```

【Note】:无法在Java代码中访问启动类加载器。`当试图获得一个类的ClassLoader时，如果得到的是null，这并不意味着没有加载器为它们服务，而是指加载这个类为启动类加载器`


在虚拟机设计中，使用分散的ClassLoader去装载类好处：不同层次的类可以由不同的ClassLoader加载，从而进行划分，有助于`系统的模块化设计`。一般来说，`启动类加载器器复制系统的核心类：rt.jar中的类；扩张类加载器用于加载%JAVA_HOME%/lib/ext/*.jar中的Java类；应用类加载器用于加载用户类，也就是用户程序的类；自定义类加载器用于架子一些特殊途径的类，一般也是用户程序类`


## ClassLoader双亲委托模式

系统中的ClassLoader在协同工作时，默认会使用双亲委托模式。即`在类加载的时候，系统会判断当前类是否已经被加载，如果已经被加载，就会直接返回可用的类，否则就会尝试加载，在尝试加载时，会先请求双亲处理，如果双亲请求失败，则会自己加载`。如下类加载器源码：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name); //如果已经加载直接返回
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false); //双亲加载
                    } else {
                        c = findBootstrapClassOrNull(name); //启动类加载器加载
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name); //当前ClassLoader尝试加载

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

双亲为null有两种情况：双亲就是启动类加载器；当前加载器就是启动类加载器


【备注】：虚拟参数`-Xbootclasspath`可以修改启动ClassPath，将指定的路径缀加到启动ClassPath后，该参数指明的路径下的类，将会被启动类加载器搜索到。`当系统需要加载一个类时，会先从顶层的启动类加载器开始加载，逐层往下，直到找到该类；当判断类是否需要加载时，是从底层的应用类加载器开始判断，如果已经在应用类加载器中的类，就不会请求上层类加载器；在判断类是否已经加载时，顶层类加载器不会询问底层类加载器`


【Note】:在判断类是否加载时，应用类加载器会顺着双亲路径往上判断，直到启动类加载器，但是启动类记载器不会往下询问，这个委托路线是单向的


### 双亲委托模式的弊端

检查类是否已经加载的委托过程是单向的，这种方式虽然从结构上说比较清晰，使各个ClassLoader的职责非常明确，但是同时会带来一个问题，即`顶层的ClassLoader无法访问底层的ClassLoader所加载的类`

通常情况下，启动类加载器中的类为系统核心类，包括一个重要的系统接口，而在应用类加载器中，为应用类。安装这种模式，应用类访问系统类自然是没有问题，但是系统类访问应用类就会出现问题：`如在系统类中提供一个接口，该接口需要在应用类中得以实现，该接口还绑定一个工厂方法，用于创建该接口的实现，而接口和工厂方法都在启动类加载器中`。这时就会出现该工厂方法无法创建由应用类加载器加载的应用实现的问题，其中包括的组件有JDBC等

### 双亲委托模式的补充

在Java平台中，把核心类(rt.jar)中提供外部服务，可由应用层自行实现的接口，通常称为Service Provicer Interface 即SPI。JDK中提供的实现有ServiceLoader，具体查看[JDK1.8源码-ServiceLoader：SPI服务接口加载器](/2019/08/19/jdk1.8-source-reading-ServiceLoader/)。


其核心步骤是：

- 获取线程上下文的ClassLoader，使该ClassLoader成为一个相对共享的实例，默认情况下，上下文加载器就是应用类加载器。这样即使是在启动类加载器中的代码也可以通过`Thread.currentThread().getContextClassLoader()`这种方式访问营养类加载器中的类。如图，以上下文加载器作为中介，使得启动类加载器得以访问应用类记载器的类：

![](\img\post.img\jvm\classloader-thread-context.png)

- 读取`META-INF/services/`目录下配置的以接口全限名为文件名，内容为接口的具体实现类配置文件，返回接口的实现类或者实例化接口实现类。


### 突破双亲模式

双亲模式的类加载方式是虚拟机默认的行为，可以通过重载ClassLoader修改这种行为。如Tomcat就有自己独特的类加载顺序。如下自定义ClassLoader的实现：从指定目录下查找全限类名加载

```java
public class CustomClassLoader extends ClassLoader {

    private String fileName;


    public CustomClassLoader(String fileName) {
        this.fileName = fileName;
    }

    @Override
    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        Class<?> c = findClass(className);//查找
        if (null == c) {
            return super.loadClass(className, resolve);
        }

        return c;
    }

    @Override
    protected Class<?> findClass(String className) {
        Class<?> clazz = this.findLoadedClass(className);
        if (null == clazz) {
            String classFile = getClassFile(className); //加载
            try {
                FileInputStream inputStream = new FileInputStream(classFile);
                FileChannel fileChannel = inputStream.getChannel();

                ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

                WritableByteChannel writableByteChannel = Channels.newChannel(outputStream);
                ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
                while (true) {

                    int read = fileChannel.read(byteBuffer);
                    if (read == 0 || read == -1) {
                        break;
                    }
                    byteBuffer.flip();
                    writableByteChannel.write(byteBuffer);
                    byteBuffer.clear();
                }
                inputStream.close();
                byte[] bytes = outputStream.toByteArray();
                clazz = defineClass(className, bytes, 0, bytes.length);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }


        return clazz;
    }


    public String getClassFile(String className) {
        return fileName + className.replace(".", File.separator) + ".class";
    }

    public static void main(String[] args) throws ClassNotFoundException {
        //D:\tmp

        CustomClassLoader customClassLoader = new CustomClassLoader("D:\\tmp\\");
        Class<?> clazz = customClassLoader.loadClass("tech.dlzp.java.code.jvm.HelloWorld");
        System.out.println(clazz.getClassLoader());//tech.dlzp.java.code.jvm.CustomClassLoader@404b9385

    }
}

```

### 热替换实现

热替换是指在程序的运行过程中，不停止服务，只通过替换程序文件来修改程序的修为，其关键`要求在于服务不能中断，修改必须立即表现在正在运行的系统之中`。基本上大部分脚本语言都是天生支持热替换

对Java程序来说，如果一个类以及加载到系统中，通过修改类文件并无法让系统再来加载并重定义这个类。因此在Java中实现这以工农的一个可行方法就是运用ClassLoader。`由不同ClassLoader加载的同名类属于不同的类型，不能相互转化和兼容`

【Note】:两个不同ClassLoader加载同一个类，在虚拟机内部，会认为这2个类是完全不同的

如图热替换基本思路：
![](\img\post.img\jvm\classloader-thread-context.png)


```java
public class HotClassLoader extends ClassLoader {

    private String path;

    public HotClassLoader(String path) {
        this.path = path;
    }

    @Override
    protected Class<?> findClass(String className) throws ClassNotFoundException {
        Class<?> clazz = this.findLoadedClass(className);
        if (clazz == null) {
            String classFile = getClassFile(className);
            try {
                FileInputStream fileInputStream = new FileInputStream(classFile);
                FileChannel fileChannel = fileInputStream.getChannel();

                ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
                WritableByteChannel writableByteChannel = Channels.newChannel(byteArrayOutputStream);

                ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
                while (true) {
                    int i = fileChannel.read(byteBuffer);
                    if (i == 0 || i == -1) {
                        break;
                    }
                    byteBuffer.flip();
                    writableByteChannel.write(byteBuffer);
                    byteBuffer.clear();
                }
                fileInputStream.close();
                byte[] bytes = byteArrayOutputStream.toByteArray();
                clazz = defineClass(className, bytes, 0, bytes.length);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }

        return clazz;
    }

    public String getClassFile(String className) {
        return path + className.replace(".", File.separator) + ".class";
    }
}

public class DoopRun {

    public static void main(String[] args) throws Exception {
        while (true) {
            HotClassLoader hotClassLoader = new HotClassLoader("d:/tmp");

            Class<?> clazz = hotClassLoader.loadClass("tech.dlzp.java.code.jvm.HelloWorld");

            Object instance = clazz.newInstance();
            Method hello = instance.getClass().getMethod("hello");

            hello.invoke(instance);
            Thread.sleep(1000);
        }
    }
}
```
























