---
layout: post
title: "Java方法中检查参数的有效性"
date: 2019-11-22 08:38:00
categories: Java 
tags: Effective-Java Java-Coding
---

* content
{:toc}

大多数方法和构造器对于传递给它们的参数值都会有某些限制，如索引值必须是非负数，对于引用不能为null等。此时就应该在API文档中清除地指明这些限制并且在方法体的开头处检查参数，以强制施加这些限制。如果传递无效参数值给方法，这个方法在执行之前先对参数进行了检查，那么它很快就会失败并且清楚地出现适当的异常(Exception)；如果没有检查参数，就有可能发生如下几种情形：

- 方法在处理过程中失败，并且产生令人费解的异常

- 方法可以正常返回，只是计算出错误的结果或者使得某个对象处理被破坏的状态(将来在某个不确定的时间，在某个不相关的点上引发错误。即没有验证参数的有效性，可能违背失败原子性`failure atomicity`)




对于public和protected的方法，要用Javadoc的@throws标签(tag)在文档中说明违反参数值限制时会抛出的异常。这样的异常通常为IllegalArgumentException、IllegalArgumentException或NullPointerException。

在Java7中Objects.requireNonNull方法进行null检查: 方法返回其输入(在使用一个值的同时执行null检查或单独检查null)

```java
strategy = Objects.requireNonNull(strategy,"exception msg");
```

对于未被到处的方法(unexported method)，作为包的创建者可以控制整个方法将在哪些情况下被调用，因此应该确保只将有效的参数值传递进来。非公有的方法通常应该使用断言`assertion`来检查参数:

```java
private static void sort(long a[] ,int offset,int length){
	assert a != null;
	assert offset >= 0 && offset <= a.length
}
```
不同于一般的有效性检查，如果断言失败，将抛出AssertionError(如果没有启动作用，本质上也不会有成本开销，除非使用-ea|-enableassertions标记传递给Java解释器)

对于有些参数，方法本身没有用到，却被保存起来供以后使用，也需要校验这类参数的有效性，尤其是构造器(创建对象时违反了当前类的约束条件)


在方法执行它的计算任务之前，应该先检查参数这一规则也有例外: 在某些情况下，有效性的检查工作非常昂贵或者根本不切实际(有效性的检查已隐藏在计算过程中)。如Collections.sort(List): 列表中的所有对象都必须是可以相互比较的，如果这些对象不能相互比较，将抛出ClassCastException，`提前检查列表中的元素是否可以相互比较并没有多大意义`【不加选择地使用这种方法将会导致失败原子性】


有时候某些计算会隐式地执行有必要的有效性检查，如果检查不成功，就会抛出错误的异常: 无效的参数值而导致计算过程的异常与文档中表明这个方法抛出的异常并不相符。在这种情况下，应该使用`异常转换(exception translation)`，将计算过程中抛出的异常转为正确的异常


总结:

- 在设计方法时，应该使方法可能通用，并符合实际的需要。假如方法对于它能接受的所有参数值都能够完成合理的工作时，对参数的限制就应该越少越好。`通常情况下，有有些限制对于被实现的抽象来说是固有的`

- 编写方法或者构造器时，应考虑参数有哪些限制条件，`把这些限制条件写入文档，并在方法体开头通过显式的检查来实施`










