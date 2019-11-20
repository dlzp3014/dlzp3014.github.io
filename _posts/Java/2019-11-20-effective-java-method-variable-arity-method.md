---
layout: post
title: "Java方法中慎用可变参数"
date: 2019-11-20 08:38:00
categories: Java 
tags: Effective-Java Java-Coding
---

* content
{:toc}

可变参数方法一般称为`variable arity method`(可匹配不同长度的变量方法)，接受0个或者多个指定类型的参数。`可变参数机制首先会创建一个数组，数据的大小为在调用位置所传递的参数变量，然后将参数值传到数组中，最后将数组传递给方法`。如下可变参数方法:

```java
public static int sum(int ... args){
	int sum = 0;
	for (int arg : args) {
		sum += arg;
	}
	return
}
```





有时间必须需要编写一个或多个某种类型参数的方法，而不是需要0个或者多个，此时就需要在运行时检测数组长度`args.length==0;throw new IllegalArgumentException("Too few arguments")`。如果客户端调用这个方法时，并未有传递参数进去(null)，此时就会在运行时而不是编译时发生异常。有一种更好的方法可以实现:`声明该方法带有两个参数，一个是指定类型的正常参数，另一个是这种类型的可变参数`

```java
public static int sum(int firstArg, int ... args){
	int sum = firstArg;
	for (int arg : args) {
		sum += arg;
	}
	IllegalArgumentException
	return
}
```

当需要让一个方法带有不一定数量的参数时，可变参数就非常有效。在重视性能的情况下，使用可变参数机制要特别小心:`每次调用可变参数方法都会导致一次数组分配和初始化`。如果凭经验无法承受这一成本但又需要可变参数的灵活性，还有一种模式可以被使用:假设确定对某个方法95%的调用会有3个或者更少的参数，就声明该方法的5个重载，每个重载方法带有0到3个普通参数，当参数的数目超过3个时，就使用一个可变参数方法:

```java
public void foo(){}
public void foo(int a1){}
public void foo(int a1, int a2){}
public void foo(int a2, int a2, int a3){}
public void foo(int a2, int a2, int a3 , int rest){}
```

当参数的数目超过3个时，所有调用中只有5%需要创建数组。就像大多数的性能优化一样，这种方法通常不太恰当，但是一旦真正需要它时，就帮上大忙。`EnumSet`类的静态工厂方法`public static <E extends Enum<E>> EnumSet<E> of()`就使用了这种方法，最大限度地减少创建枚举集合的成本(因为枚举集合为位域提供了在性能方面有竞争力的替代方法)。


总结：

- `在定义参数数目不定的方法时，可变参数方法是一种很方便的方式`

- `在使用可变参数之前，要先包含所有必要的参数，并且要关注使用可变参数所带来的性能影响`





