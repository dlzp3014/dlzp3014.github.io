---
layout: post
title:  "Java中锁在应用层的优化"
date:   2019-08-25 08:38:00
categories: Java 
tags: JVM Concurrent
---

* content
{:toc}


在实际软件开发过程中，如果在应用层能合理地进行锁的优化，对系统性能也有积极作用。常见的方法如下：减少锁持有时间、减少锁粒度、锁分离、锁粗化、无锁(CAS)






## 减少锁的持有时间

对于使用锁进行并发控制的应用程序而言，在锁竞争过程中，`单个线程对锁的持有时间与系统性能有着直接的关系`。如果`线程持有锁的时间很长，相对锁的竞争程度就越激烈`。因此，`在程序开发过程中，应该尽可能地减少对某个锁的占有时间，以减少线程间互斥的可能`。如下代码片段：

```java
public synchronized void syncMethod(){
	doSome(); //非同步控制方法
	mutexMethod();//互斥方法，
}
```
如果doSome()方法需花费较长的CPU时间，此时，如果在并发量较大时，使用这种`对整个方法做同步的方法，会导致等待线程大量增加`，其原因在于`一个线程在进入该方法时获得内部锁，只有在所有任务都执行完后，才会释放锁`。一个较为优化的解决方案是只在`必要时进行同步`，这样就能明显减少线程持有锁的时间，提供系统的吞吐量:

```java
public void syncMethod(){
	doSome(); //非同步控制方法，执行时间长
	synchronized(this){ //只针对非线程安全方法做同步，锁占用的时间相对较短，因此能有更高的并行度
		mutexMethod();//互斥方法，
	}
}
```

减少锁的持有时间这种锁的优化方式在JDK的源码包中可以很容易地找到，如处理正则表达式的Pattern类：

```java
public Matcher matcher(CharSequence input) {
    if (!compiled) {
        synchronized(this) {
            if (!compiled)
                compile();
        }
    }
    Matcher m = new Matcher(this, input);
    return m;
}
```

matcher方法有条件地进行锁的申请：只有在表达式未编译时，进行局部的加锁，这种处理方式大大提高了matcher()方法的执行效率和可靠性

【Note】:减少锁的持有时间有助于`降低锁冲突的可能性`，进而提升系统的并发能力


## 减少锁粒度

减少锁粒度是一种`削弱多线程锁竞争的有效手段`。这种技术典型的使用场景就是ConcurrentHashMap类的实现。对于一个普通的集合对象的多线程同步来说，最常使用的方式就是对get()和add()方法进行同步：每当对集合进行add()操作或者get()操作时，总是获得集合对象的锁，因此，`事实上没有两个线程可以做到真正的并发，任何线程在执行这些同步方法时，总要等待前一个线程执行完毕`，在高并发时，激烈的锁竞争会影响系统的吞吐量。作为JDK并发包重要成员ConcurrentHashMap类，很好地使用了拆分锁对象的方式提高ConcurrentHashMap的吞吐量。

`ConcurrentHashMap将整个HashMap分成若各段(Segment)，每个段都是一个子HashMap。如果需要在ConcurrentHashMap中添加一个新的项，并不是将整个HashMap加锁，而是首先根据hashcode得道该项应该被存放到哪个段中，然后对该段加锁，并完成put()操作。在多线程环境中，如果多个线程同时进行put()操作，只要被加入的项不存在同一个段中，则线程间便可以做到真正的并行`。默认情况下，ConcurrentHashMap拥有16个段


减少锁粒度会引入一个新的问题：`当系统需要取得全局锁时，器消耗的资源会比较多`。如ConcurrentHashMap，虽然其put()方法很好的分离了锁，但是当试图访问ConcurrentHashMap`全局信息`时，就会需要同时`取得所有段`的锁方能顺利实施，如size()方法，它将返回ConcurrentHashMap的有效项的数量，即ConcurrentHashMap(JDK1.7)的全部有效项之和，要获得这个信息需要取得所有字段的锁。


【Note】:减少锁粒度，就是指缩小锁定对象的范围，从而减少锁冲突的可能性，进而提高系统的并发能力



## 锁分离

锁分离是减小锁粒度的一个特例，它依据应用程序的功能特点，`将一个独占锁分成多个锁`。一个典型的案例就是LinkedBlockingQueue的实现

在LinkedBlockingQueue的实现中，take()和put函数分别实现了从队列中取得数据和往队列中添加数据的功能。虽然两个函数都对当前队列进行了修改操作，但是由于LinkedBlockingQueue是基于链表的，因此。两个操作分别作用于队列的前端和后端，从理论上说两种并不冲突；如果使用独占锁，则要求在两个操作进行时它们会彼此等待对方释放锁资源，在这种情况下，锁竞争相对会比较激烈，从而影响程序在高并发时的性能。具体实现过程请参看[JDK1.8源码-LinkedBlockingQueue阻塞队列](/2019/08/27/jdk1.8-source-reading-LinkedBlockingQueue/)


## 锁粗化

通常情况下，为了保证多个线程键的有效并发，会要求每个线程持有锁的时间尽量短，即在使用完公共资源后，应该立即释放锁，这样才能让等待在这个锁的其他线程尽早地获得资源执行任务。但是，如果`对同一个锁不停地进行请求、同步和释放`，其本身也会消耗系统的资源，反而布列于性能的优化。为此，·虚拟机在遇到一连串地对同一锁不断进行请求和释放的操作时，便会把所有的锁操作整合成对锁的一次请求，从而减少对锁的请求同步次数，这个操作就叫做锁的粗化。如下代码段：

```java
public void desome(){

	synchronized(lock){
		//do some
	}
	synchronized(lock){

	}
}
```
其将会别整合为：

```java
public void desome(){

	synchronized(lock){
		//do some
	}
	
}
```

在开发过程中，应该有意识地在`合理的场合进行锁的初始化`,尤其在循环内请求锁时：

```java
for(;;){
	//对锁进行大量的请求
	synchronized(lock){

	}
}
```

以上代码在每个循环时，都对同一个对象申请锁。此时应该将其锁粗化为：

```java
synchronized(lock){ //只进行一次锁请求
	for ( ; ; ) {
		
	}

}

```


## 总结

`性能优化就是根据运行时的真实情况对各个资源点进行权衡折中的过程`。锁粗化和减少锁的持有时间是相反的，但是在不同的场合它们的效果并不相同，开发人员需要根据实际情况进行权衡。此外，虚拟机内部的锁优化策略偏向锁、自旋锁等，它们也不是绝对地可以提供系统性能，对锁的优化，还需要做更多的权衡和思考




