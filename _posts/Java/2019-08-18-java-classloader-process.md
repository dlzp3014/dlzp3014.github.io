---
layout: post
title:  "Class文件的装载流程"
date:   2019-08-18 08:38:00
categories: Java 
tags: Java-JVM
---

* content
{:toc}

Class类型通常以文件的形式存在(任何二进制流都可以是Class类型)，只有被Java虚拟机装载的Class类型才能在程序中使用。系统装载Class类型可以分为加载、连接和初始化3个步骤，连接又可以分为验证、准备和解析3步：




## 类装载的条件

`Class只有在必须要使用的时候才会被装载，Java虚拟机不会无条件地装载Class类型`。Java虚拟机规定，一个类或者接口在初始使用前，必须要进行初始化。这里的使用指主动的使用，主动使用只有下列几种情况：

- 创建一个类的实例时，如使用new关键字或者通过反射、克隆、反序列化

- 调用类的静态方法时，即使用了字节码invokestatic指令

- 使用类或接口的静态字段时(final常量除外)，如使用getstatic或者putstatic指令

- 使用java.lang.reflect包中的方法反射类的方法时

- 初始化之类时，要求先初始化父类

- 作为启动虚拟机含有的main()方法的类

除以上情况属于主动适应外，其他的情况均属于被动使用，被动使用不会引起类的初始化

如下主动引用：

```java
public class Main {

	public static void main(String[] args) {
		new Child(); 
		/**
		 * print: 
		 * init parent...
		 * init child ...
		 */
	}
}

class Parent {
	static { //类初始化时指定static代码块
		System.out.println("init parent...");
	}
}

class Child extends Parent {
	static {
		System.out.println("init child ...");
	}
}
```

系统首先装载Parent类，接着在装载Child类。符合主动装载中的两个条件：`使用new关键字创建类的实例会装载相关类，以及在初始化子类的时候，必须先初始化父类`

如下被动引用不会导致类的装载：

```java
public class Main {

	public static void main(String[] args) {
		System.out.println( Child.value);  //Child类并未被初始化
		/**
		 * print: 
		 * init parent...
		 * 10
		 */
	}
}

class Parent {
	static {
		System.out.println("init parent...");
	}
	
	public static int value=10;
}

class Child extends Parent {
	static {
		System.out.println("init child ...");
	}
}
```

从执行结果来看，虽然main()方法中直接访问了子类对象，但是Child子类闭关未被初始化，只有parent父类被初始化。`可见在引用一个字段时，只有直接定义该字段的类才会被初始化`

【Note:】虽然Child类没有被初始化，但是此时`Child类已经被系统加载，只是没有进入初始化阶段。也就说明类，`类被加载进来也并不一定会执行初始化操作`使用-XX:+TraceClassLoading`参数运行时，就可以得到具体类加载的过程：

```java
[Loaded Main from file:/D:/workspace/test/bin/]
[Loaded java.lang.Void from shared objects file]
[Loaded Parent from file:/D:/workspace/test/bin/]
[Loaded Child from file:/D:/workspace/test/bin/] # Child类已经被加载，但是并未进行初始化
init parent...
10
[Loaded java.lang.Shutdown from shared objects file]
```

如下引用final常量并不会引起类的初始化

```java
public class FinalFieldClass {
    public static final String HELLO="hello";
    static {
    	//并不会执行初始化方法
        System.out.println("FinalFieldClass init");
    }
}

public class UseFinalField {

    public static void main(String[] args) {
        System.out.println(FinalFieldClass.HELLO);
    }
}
```

FinalFieldClass类并没有因为其常量字段value被引用而初始化。这是因为在Class文件生成时，final常量由于其不变性，做了适当的优化。查看main方法的字节码：

```java
0 getstatic #2 <java/lang/System.out>
3 ldc #4 <hello>
5 invokevirtual #5 <java/io/PrintStream.println>
8 return
```

在字节码偏移3的位置，通过`ldc`将常量池第4项入栈，在此Class文件中常量池第4项为：

```java

String cp_info#25<hello>
Length of byte arryy 5
Length of string 5
String hello
```

在编译后UseFinalField类中，并没有引用FinalFieldClass类，而是将其final常量直接放入到常量池中，因此FinalFieldClass类也不会被加载，通过捕获类加载日志也会发现并不会被加载到系统中。

`javac在编译时，将常量直接植入目标类，不再使用被引用类`


【Note】：并不是在代码中出现的类，就一定会被加载或者初始化，如果不符合主动使用的条件，类就不会初始化

## 加载类

加载类处于类装载的第一个阶段。在加载类时，Java虚拟机必须完成如下工作：

- 通过类的全名，获取类的二进制数据流

- 解析类的二进制数据流为方法区的数据结构

- 创建java.lang.Class类的实例，表示该类型




对于类的二进制流，虚拟机可以通过多种途径产生或者获得，一般地虚拟机可能通过文件系统读入一个class后缀的文件，或者也可能读入JAR、ZIP等归档数据包而提取类文件，还可以事先将类的二进制数据存放在数据库中或者类似于HTTP之类的协议通过网络进行加载，甚至是在运行时生成一段Class的二进制信息。在获得类的二进制信息后，Java虚拟机就会处理这些数据，并最终转为一个`java.lang.Class的实例`，这个是实例是访问类型元数据的接口，也是实现反射的关键数据。通过Class提供的接口，可以访问一个类型的方法，字段等信息。如使用Class.forName("fullName")[全限定类名]获取到Class对应的实例后，执行getDeclareMethods()即可得到类中的所有方法

`在Java虚拟机中，完成加载类，尤其是获取类的二进制信息的组件为ClassLoader类加载器`，具体参考[JVM-ClassLoader类加载器](/2019/08/18/java-classloader/)

## 验证类

当类加载到系统后，就开始`连接`操作，验证是连接操作的第一步，目的为保证加载的字节码是合法、合理，且符合规范。验证的步骤比较复杂，实际要验证的醒目也很繁多，大体上Java虚拟机需要做以下检测

- 格式检查：魔数检查、版本检查、长度检查

必须判断类的二进制数据是否符合格式要求和规范。如是否也魔数0xCAFEBASE开头，主版本和小版本号是否在当前Java虚拟机的支持范围内，数据中每一项是否都拥有正确的长度等

- 语义检查：是否继承final、是否有父类、抽象方法是否有实现

Java虚拟机会进行字节码的语义检查。如是否所有的类都有父类的存在(在Java中除了Object外，其他类都应该有父类)，是否一些被定义为final的方法或者类被重载或者继承了，非抽象类是否实现了所有抽象方法或者接口方法，是否存在不兼容的方法(方法的签名除了返回值不同，其他都一样，这种方法会让虚拟机无从下手调度)。但凡在语义上不符合规范的，虚拟机也不会给予验证通过

- 字节码验证：跳转指令是否指向正确的位置，操作数类型是否合理

Java虚拟机还会进行字节码验证，字节码验证也是验证过程中最为复杂的一个过程，它试图通过对字节码流的分析，判断字节是否可以被正确的执行。如在字节码的执行过程中是否会跳转到一条不存在的指令，函数的调用是否传递了正确类型的参数，变量的复制是否给定了正确的数据类型等。`栈映射帧(StackMapTable)就是在这个阶段，用于检测在特定的字节码处，其局部变量表和操作数栈是否有正确的数据类`/

100%准确地判断一段字节码是否可以被安全执行时无法实现的，此过程只是尽可能地检查出可以预知的明显问题。在这个阶段无法通过检查，虚拟机也不会正确装载这个类，但是通过了这个阶段的检查，也不能说明这个类完全没有问题


- 符号引用验证：符号引用的直接引用是否存在

校验器还将进行符号引用的验证。`Class文件在其常量池也会通过字符串记录自己将要是有的其他类或者方法`。因此，在验证阶段，虚拟机就会检查这个类或者方法确实存在，且当前类有权限访问这些数据。`如果需要使用的类无法在系统中找到，则会抛出NoClassDefFoundError，如果一个方法无法找到，则会抛出NoSuchMethodError`


## 准备

当一个类验证通过时，虚拟机就会进入准备阶段。在这个阶段，虚拟机就会为这个类分配相应的内存空间，并设置初始值。Java虚拟机为各类型变量默认的初始值如下：
|类型|默认初始值|
|---|---|
|int|0|
|long|0l|
|short|(short)0|
|char|\u0000|
|boolean|false|
|reference|null|
|floar|0f|
|dloar|0f|



【Note:】Java并不支持boolean类型，对于boolean类型，内部实现是int，由于int的默认值是0，因此对于的boolean的默认值就是false

如果类存在常量字段，常量字段也会在准备阶段被赋予正确的值，这个赋值属于Java虚拟机的行为，属于变量的初始化。在准备阶段不会有任何Java代码被执行

如在类中定义如下常量：
```java
public static final String HELLO="hello_world";

```
在生成的Class文件，就可以看到该字段还有HELLO属性，直接存放与常量池中，该常量在准备阶段被附上字符串"hello_world"(并非由Java字节码引起)

![](/img/post.img/jvm/constant-holle.png)

如果没有final的修饰，仅仅作为普通的静态变量：

```java
public static String hello="hello_world";

```

此时hello的赋值在函数<clinit>中发生，属于Java字节码的行为。字段hello上未携带任何数据信息，在<clinit>方法中，将字符串常量hello_world通过ldc指令压栈，并通过putsatic语句进行赋值

`ldc字节码会加载一个常量到操作数栈中，putstatic字节码设置给定的静态字段的值`

## 解析类

在准备阶段完成后，就进入解析阶段。解析阶段的工作就是将类、接口、字段和方法的符号引用转为直接引用。`符号引用是一些字面量的引用，和虚拟机的内部数据结构和内存布局无关`。容易理解的就是在Class文件中，通过常量池进行了大量的符号引用。如下说明符号引用的工作原理：

![](/img/post.img/jvm/chart-reference.png)

常量池#6项被invokevirtual使用，通过查看CONSTANT_Methodref #6的引用关系，最终发现，所由对于Class以及NameAndType类型的引用都是就字符串的。因此，可以认为invokevirtual的函数调用用过字面量的引用描述已经表达清楚，这就是符号引用


在程序实际运行时，只有符号引用是不够的，当println方法被调用时，系统需要明确知道该方法的位置。Java虚拟机为每一个类都准备一张方法表，将其所有的方法都列在表中，当需要调用一个类的方法的时候，只要知道这个方法在方法表中的`偏移量`就可以直接调用该方法。`通过解析操作`，符号引用就可以转变为目标方法在类中方法表中方法表的位置，从而使得方法被成功调用


`所谓解析就是将符号引用转为直接引用，也就是得到类或字段、方法在内存中的指针或者偏移量。因此，如果直接引用存在，那么可以肯定系统中存在该类、方法或者字段，但只存在符号引用，不能确定系统中一定存在该对象`


CONSTANT_String解析：由于字符串在程序开发中有着重要的作用，因此了解String在Java虚拟机中的处理非常重要。当在Java代码中直接使用字符串常量时，就会在类中出现CONSTANT_String，表示字符串常量，并且会引用一个CONSTANT_UTF8的常量项。在Java虚拟机内部运行时的常量池中，会维护一张字符串`拘留表`，它会保存所有出现过的字符串常量，并且没有重复项。只要以CONSTANT_String形式出现的字符串也都会在这张表中。`使用String.inter()方法可以得到一个字符串在拘留表(intern)中的引用`。因为该表中没有重复项，所以任何字面相同的字符串的String.intern()方法返回总是相等的

```java
String a = "123";
String b = "1" + "2" + "3";
System.out.println(a.equals(b)); //true:字面量
System.out.println(a==b);//false：指向同一个对象引用
System.out.println(a.intern()==b);//true：拘留表intern中的引用就是b常量本身
```


## 初始化

类的初始化时类装载类的最后一个阶段，前面的步骤都没有问题时，表示类可以顺利装载在系统中，此时类才开始执行Java字节码。初始化阶段的重要工作时执行类的初始化方法`<clinit>`，这个方法是又编译器自动生成的，它`由类静态成员的赋值语句已经static语句块合并产生的`

```java

public class UseFinalField {

    public static int id = 1;

    public static String name;

    static {
        name = "dlzp";
    }
}
```

```java
0 iconst_1
1 putstatic #2 <tech/dlzp/java/code/jvm/UseFinalField.id>
4 ldc #3 <dlzp>
6 putstatic #4 <tech/dlzp/java/code/jvm/UseFinalField.name>
9 return

```

在生成的<clinit>函数中，整合了UseFinalField类中的static赋值赋予已经static语句块，先后对id和name两个成员变量进行赋值


`在加载一个类之前，虚拟机总是会尝试加载给类的父类，因此父类的<clinit>总是在子类的<clinit>之前被调用，子类的static块优先级高于父类`

`Java并不会为所有的类都产生<clinit>初始化函数：类既没有赋值语句，也没有static语句块，那么生成的<clinit>函数就应该为空，编译器就不会为该类插入<clinit>函数`。如类中只有final常量，final常量在准备阶段初始化，并不在初始化阶段处理，因此在产生的class文件中，没有<clinit>函数


【Note:】对于<clinit>函数的调用，也就是类的初始化，`虚拟机在内部确保其多线程环境中的安全性`。也就是说，当多个线程尝试初始化同一个类时，只有一个线程可以进入<clinit>函数，而其他线程必须等待，如果之前的线程成功加载了该类，则等在队列中的线程就没有机会再执行<clinit>函数(当需要使用这个类时，虚拟机会直接返回给它已经准备好的信息)。正是因为<clinit>函数是带有锁线程安全的，因此，`在多线程环境下进行类初始化的时候，可能会引起死锁，且这种死锁是很难发现的(看不到任何可用的锁信息)`

```java

public class ClassClinitDeadLock implements Runnable {

    //tech.dlzp.java.code.jvm.B
    String str;

    public ClassClinitDeadLock(String str) {
        this.str = str;
    }

    public static void main(String[] args) {
        Thread a = new Thread(new ClassClinitDeadLock("A")); //尝试去初始化A:在A初始化过程中，去尝试初始化B
        a.start();
        Thread b = new Thread(new ClassClinitDeadLock("B"));//尝试去初始化B:在B尝试化过程中，初始化A
        b.start();
    }

    @Override
    public void run() {
        try {
            Class.forName("tech.dlzp.java.code.jvm." + str);
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        System.out.println("run ....");
    }

}

class A {

    static {
        try {
            Thread.sleep(1000);
            Class.forName(B.class.getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        System.out.println("A static init ok");
    }
}

class B {
    static {
        try {
            Thread.sleep(1000);
            Class.forName(A.class.getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        System.out.println("B static init ok");
    }
}
```









