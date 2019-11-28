---
layout: post
title: "Java中抛出与抽象对应的异常: 异常转译"
date: 2019-11-28 08:38:00
categories: Java 
tags: Effective-Java Java-Coding
---

* content
{:toc}

如果方法抛出的异常与它所执行的任务没有明显的联系时，这种情况将会使开发者不知所措，这种情况常常发生在由`底层抽象抛出的异常`。为避免这个问题，更高层的实现应该捕获底层的异常，同时抛出可以按照高层抽象进行解释的异常，这种做法被称为异常转译`exception translation`，代码如下:

```java
try{

}catch(LowerLevelException e){
	throw new HigherLevelException(...);
}
```



一种特殊的异常转译形式称为异常链`exception chaining`，如果`底层的异常对于调试高层异常的问题非常有帮助`，使用异常链就很合适: 底层异常原因被传递到高层异常，高层的异常提供访问方法(Throwable#getCause方法)来获取底层的异常:

```java
try{

}catch(LowerLevelException cause){
	throw new HigherLevelException(cause);
}
```

高层异常的构造器将原因传到支持链(chaining-aware)的父构造器，因此它最终将被传给Throwable运行异常链的构造器中。`大多数标准的异常都支持链的构造器，对于没有支持链的异常，可以利用Throwable的initCause方法设置原因`。异常链不仅可以通过程序的getCause方法原因，还恶意将原因的堆栈轨迹集成到更高层的异常中


尽管异常转译与不加选择地从底层传递异常的做法相比有所改进，但是也不能滥用: 

处理来自底层异常的最好做法是，`在调用底层方法之前确保它们会执行成功`，从而将高层方法的调用者与底层的问题`隔离`开来。这种情况下，可以用某种适当的记录机制将异常记录下来(Logging)，这样有助于排查问题，同时又将客户端代码和最终用户与问题隔离开来


总结: 

- 如果不能阻止或者处理来自更底层的异常，一般的做法是使用异常转译，只有在底层方法可以保证它所抛出的所有异常对于更高层也是合适的情况下，才可以将异常从底层传播到高层

- 异常链对于高层和底层异常都提供了最佳的功能: 允许抛出适当的高层异常，同时又能捕获底层的原因进行失败分析

