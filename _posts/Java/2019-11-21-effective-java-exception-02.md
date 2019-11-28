---
layout: post
title: "Java中对可恢复的情况使用受检查异常、对编程错误使用运行时异常"
date: 2019-11-21 08:38:00
categories: Java 
tags: Effective-Java Java-Coding
---

* content
{:toc}

Java中提供了三种可抛出`throwable`异常: 受检查异常(checked exception)、运行时异常(runtime exception)和错误(error)。关于什么时候使用哪种可抛出的异常，可以使用如下指导原则:

- `期望调用者能够适当的恢复，就应该使用受检查异常`

如果抛出受检查异常，强迫调用者在一个catch子句中处理该异常或者将它们转播出去。因此，方法中声明要抛出的每个受检查异常，都是对API用于的一种潜在提示: 于异常相关联的条件是调用这个方法的一种可能结果。API的设计者让API用于面对受检查异常，以此来强制用户从这个异常条件中恢复，用户可以忽略这样的强制要求(只需捕获异常并忽略即可[`不推荐`])。




运行时异常和错误都是未受检查异常，在行为上两者是等同的: `都不需要也不应该被捕获的异常`。如果程序抛出未受检查的异常或者错误，往往就属性不可恢复的情形，继续执行下去有害无益。如果程序没有捕获到这样的可抛出异常，将会导致当前线程中断，并出现适当的错误信息


- 编程错误使用运行时异常来表明

大多数的运行时异常都表示前提违例(precodition violation): 是指API的用户没有遵守API规范建立的约定，如数组访问的约定指明了数组的下标必须在零和数组长度-1之间(ArrayIndexOutOfBountsException表明违反了这个前提)

对于要处理可恢复的条件，还是处理编程错误，情况并非总是分明：如考虑资源枯竭的情形，这可能是由于程序错误引起的(分配不合理的过大数，也可能确实是由于资源不足而引起的)，如果资源枯竭是由于临时短缺或是临时需求太大所造成，这种情况可能就需要恢复。因此API设计时就需要判断这样的资源枯竭是否允许恢复，`如果可能允许恢复就使用受检查异常，否则就使用运行时异常。如果不清楚是否有可能恢复，最好使用未受检查异常`


- 不要在实现任何新的Error子类

错误经常被JVM保留下来使用，以表明资源不足、约束失败或者其他使程序无法继续执行的条件，因此`最好不要在实现任何新的Error子类`，实现的所有未受检查的异常都应该是RuntimeExcpetion的子类(直接或者间接)，不应该定义Error子类，甚至也不应该抛出AssertionEError异常。JLS并没有直接规定这样的异常结果，而是指定其从行为意义上讲等同于普通的受检查异常(Exception子类，而不是RuntimeException)



异常也是一个完整意义上的对象，可以在其定义任意的方法: 主要用于捕获异常的代码而提供额外的信息，特别是关于引发这个异常条件的信息。如果没有这些信息方法，程序员必须懂得如何解析`该异常的字符串表示法`以便获得这些额外的信息。

受检查异常往往指明了可恢复的条件，所以对于这样的异常，需要提供一些辅助的方法可以帮助调用者获得一些有助于恢复的信息。如用户资金不足时可以抛出一个受检查一次，这个异常应该提供一个访问方法以便用户查询所缺的费用金额，使得调用者可以将这个数字传递给用户

总结: 对于可恢复的情况，要抛出受检查一次；对于程序错误，要抛出运行时异常；不确定是否可恢复，则抛出未受检查异常。不要定义任何既不是受检查异常也不是运行时异常的抛出类型。要在受检查异常上提供方法，以便协助恢复




