---
layout: post
title:  "JVM中锁的实现和优化"
date:   2019-08-12 08:38:00
categories: Java 
tags: Java JVM Concurrent
---

* content
{:toc}

在多线程程序中，线程之间的竞争是不可避免的，因此如何使用更高的效率处理多线程的竞争是JVM的一项重要任务。如果将所有的线程竞争都交由操作系统处理，那么并发性能将非常低。为此，`JVM在操作系统层面挂起线程之前会先尽可能在虚拟机层面上解决竞争关系，尽可能的避免真实的竞争发生`。同时在竞争不激烈的场合，也会`视图消除不必要的竞争`。`JVM实现这种手段的方法主要包括：偏向锁、轻量级锁、自旋锁、锁消除、锁膨胀等`。其中涉及到的对象Mark Word请参看[JVM中对象头和锁](/2019/08/22/java-jvm-markword-lock/)




## 偏向锁

偏向锁是JDK1.6提出的一种锁优化方式，核心思想是`如何程序没有竞争，则取消之前已经取得锁的线程同步操作`。也就是说，若`某一个锁被线程获取后，遍进入偏向模式，当线程再次请求这个锁时，无需再进行相关的同步操作，从而节省操作时间，如果在此之间有其他线程进行了锁请求，则退出偏向模式`。在JVM中使用`-XX:+UserBiasedLocking`可以设置启用偏向锁

【Note】:偏向锁其核心也就是无其他线程获取锁时，取消同步操作

当锁对象处于偏向模式时，对象头会记录获得锁的线程：

```java
[JvaThread* | epoch | age| 1 |01]

```
`当该线程再次尝试获得锁时，通过Mark Word的线程信息就可以判断当前线程是否持有偏向锁`


执行Java程序时，可以使用以下参数: `-XX:+UserBiasedLocking -XX:BiasedLockingStartupDelay=0` ，其中BiasedLockingStartupDelay表示虚拟机在启动后，立即启用偏向锁，如果不设置该参数，默认虚拟机会在启动后4秒才启用偏向锁


【Note】:偏向锁在锁竞争激烈的场合没有太强的优化效果，因为`大量的竞争会导致持有锁的线程不停的切换，锁也很难一直保持在偏向模式`。此时，如果`使用偏向锁不仅得不到性能的优化，反而有可能降低系统性能`。因此，在激烈竞争的场合，可以尝试使用`-XX:-UserBiasedLocking`参数禁用偏向锁


## 轻量级锁

`如果偏向锁失败，Java虚拟机会让线程申请轻量级锁`。轻量级锁在虚拟机内部使用一个称为BasicObjectLock的对象实现，这个对象内部由一个BasicLock对象和一个持有该锁的Java对象指针组成。BasicObjectLock对象放置在JJava栈的栈帧中，在BasicLock对象内部还维护着displaced_header字段，用于备份对象头信息的Mark Word

当一个线程持有一个对象的锁时，对象头部Mark Word如下所示:

```java
[ptr | 00 ] locked
```
末尾两位为00，整个Mark Word为指向BasicLock对象的指针。由于BasicObjectLock堆栈在线程栈中，因此该指针必然指向持有该锁的线程栈空间。当需要判断某一线程是否持有该对象锁时，也只需简单地判断对象头的指针是否在当前线程的栈地址范围内即可。同时BasicLock对象的displaced_header字段，备份了原对象的Mark Word内容。BasicObjectLock对象的obj字段则指向该对象。


在虚拟机的实现中，关于轻量级锁的实现过程大体如下：

- 首先，BasicLock通过set_displaced_header(mark)备份原对象的Mark Word

- 接着，使用CAS操作尝试将BasicLock的地址复制到对象头的Mark Word。如果复制成功，那么加锁成功，否则认为加锁失败。如果加装失败，则轻量级锁就可能被膨胀为重量级锁

轻量级锁示意图：

![](\img\post.img\jvm\light-weight-lock.png)


## 锁膨胀/重量级锁

当轻量级锁失败，虚拟机就会使用重量级锁。在使用重量级锁时，对象的Mark Word如下：

```java
[ptr |10] monitor
```

末尾的2位被设置为10。整个Mark Word表示指向monitor对象的指针。在轻量级锁处理失败后，虚拟机执行以下操作：

- 废弃前面BasicLock备份的对象头信息
- 正式启用重量级锁：

首先通过inflate进行锁膨胀，目的是获得对象的ObjectMonitor;然后使用enter方法尝试进入该锁，在其方法调用中，`线程可能会在操作系统层面被挂起，如果这样，线程键切换和调度成本就会比较高`

## 自旋锁

锁膨胀后，进入ObjectMonitor的enter方法，线程可能会在操作系统层面被挂起，这样线程上下文切换的性能损失就比较大。因此，在锁膨胀之后，虚拟机会做最后的争取，希望`线程可以尽快进入临界区而避免被操作系统挂起`。一种较为有效的手段就是使用自旋锁

`自旋锁可以使线程在没有取得锁时，不被挂起，而转而去执行一个空循环(所谓的自选)，在若干个空循环后，线程如果可以获得锁，则继续执行。若选出依然不能被获得锁，才会被挂起`。使用自旋锁后，`线程被挂起的几率相对减少，线程执行的连贯性相对加强`。因此对于那些`竞争不是很激烈，锁占用时间很短的并发线程`，具有一定的积极意义，但对于锁竞争激烈，但线程锁占用时间长的并发程序，自旋锁在自旋等待后，往往依然无法获得对应的锁，不仅仅浪费了CPU时间，最终还是免不了执行被挂起的操作,反而浪费了系统资源

JDK1.6中，Java虚拟机提供-XX:+UseSpinning参数来开启自旋锁，使用-XX：PerBlockSpin参数来设置自旋锁的等待次数

JDK1.7中，自旋锁的参数被取消，虚拟机不再支持由用户配置自旋锁，自旋锁总是会执行，自选次数也由虚拟机自行调整


## 锁消除

锁消除是`Java虚拟机在JIT(Just-In-Time:实时编译)编译时，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁`。通过锁消除，可以节省毫无意义的`请求锁时间`。

【Note】：如果不可能存在竞争，为什么程序员还要加上锁，这时因为在Java软件开发过程中，开发人员必然会使用一个JDK的内置API,如StringBuffer等，这些常用的工具类可能被大面积地使用。虽然这些工具类本身可能有对应的非线程安全版本，但是开发人员也很有可能在`完全没有线程竞争的场合使用它们`。在这种情况下，这些工具类内部的同步方法就是不比较的。虚拟机可以在运行时，基于`逃逸分析技术，捕获到这些不可能存在竞争却有申请锁的代码段，并消除这些不必要的锁`,从而提高系统性能。如下StringBuffer代码段：

```java
public static String creatBs(String s1,String s2){
	StringBuffer sb=new StringBuffer();
	sb.append(s1);
	sb.append(s2);
	return sb.toString();
}

```

逃逸分析和锁消除分别可以使用参数-XX:+DoEscapeAnalysis和-XX:+EliminateLocks开启(锁消除必须工作在-server模式下)

如下使用以下参数调用当前方法：

```java
-server -XX:+DoEscapeAnalysis -XX:-EliminateLocks -Xcomp -XX:-BackgroundCompilation -XX:BiasedLockingStartupDelay=0
```
上述参数关闭了锁消除，因此每次append()操作都会进行锁的申请。如果开启锁消除，即使用以下参数：

```java
-server -XX:+DoEscapeAnalysis -XX:+EliminateLocks -Xcomp -XX:-BackgroundCompilation -XX:BiasedLockingStartupDelay=0 //BiasedLockingStartupDelay 迫使偏向锁在启动时就生效

```

【偏向锁与锁消除】：偏向锁本身简化了锁的获取，即便如此，性能也不如锁消除后的代码











