---
layout: post
title:  "Java中的Bridge Method桥接方法"
date:   2019-10-20 08:38:00
categories: Java 
tags: Java-Reflect
---

* content
{:toc}

桥接方法是JDK1.5引入泛型后，为了使Java的泛型方法生成的字节码和1.5版本前的字节码相兼容，由编译器自动生成的方法。可以通过Method#isBridge()来进行判断




## 桥接方法

Bridge method是Java里的一个概念，正常情况下使用得并不多，在Java Language Specification规范中有详细的介绍[15.12.4.5. Create Frame, Synchronize, Transfer Control ](https://docs.oracle.com/javase/specs/jls/se7/html/jls-15.html#jls-15.12.4.5)，举例如下：



``` java
abstract class C<T> {
    abstract T id(T x);
}
class D extends C<String> {
    String id(String x) { return x; }
}

C c = new D();
c.id(new Object());  // fails with a ClassCastException
```

在C类中定义了泛型，子类中给泛型设置了String，使用时新建了D类型的C，调用时由于C中泛型没有指定具体的类型，所以可以给它的id方法传递任意类型的参数，上面代码中传入Object编译并没有问题，但实际使用的是D类型的实例，D给方法设置了String，这在运行时就出错了。

Java需虚拟机中会给D创建两个id方法，处理自定义的String为参数的id方法，还会创建一个Object做参数的方法，创建的方法如下：

```java
Object id(Object x) { return id((String) x); }
```

这个Object为参数的方法就叫桥接方法(Bridge Method)，它作为一个桥将Object为参数的调用转换到了String为参数的方法，目的在于兼容1.5 版本前的字节码

## 桥接方法生成时机

`一个子类在继承（或实现）一个父类（或接口）的泛型方法时，在子类中明确指定了泛型类型，那么在编译时编译器会自动生成桥接方法`。如下生成的字节码文件：

```java
abstract class Transform<T> {
    abstract String convert(T t);

}

class StringTransform extends Transform<String> {

    String convert(String s) {
        return s;
    }
}
```
![](/img/post.img/jvm/bridge-method.png)

StringTransform只声明了一个convert方法，但是从字节码文件可以看出有两个convert方法：conver(String s) 和conver(Object s)，其中conver(Object s)方法就是编译器自动生成的桥接方法：参数类型和返回值类型都是Objec，把Object类型的参数强制转换成了String类型(`类型不匹配时就会出现ClassCastException异常`)，再调用在StringTransform类中声明的方法


## 桥接方法作用

在java1.5以前，比如声明一个集合类型：

```java
List list = new ArrayList();
```
此时就可以向list中添加任何类型的对象，但是在从集合中获取对象时，无法确定获取到的对象是什么具体的类型，所以在1.5的时候引入了泛型，在声明集合的时候就指定集合中存放的是什么类型的对象：

```java
List<String> list = new ArrayList<String>();
```

此时在获取时就不必担心类型的问题，因为`泛型在编译时编译器会检查往集合中添加的对象的类型是否匹配泛型类型`，如果不正确会在编译时就会发现错误，而不必等到运行时才发现错误。

泛型是在1.5引入的，`为了向前兼容，所以会在编译时去掉泛型（泛型擦除）`，由于java泛型的擦除特性，如果不生成桥接方法，那么与1.5之前的字节码就不兼容了


