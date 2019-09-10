---
layout: post
title:  "静态工厂方法命名习惯"
date:   2019-09-10 08:38:00
categories: Java 
tags: Coding
---

* content
{:toc}


- from:类型转换方法

只有一个参数，返回该类型的一个相对应的实例

```java
Date date=Date.from(instant) ;
```




- of:聚合方法，带有多个参数，返回该类型的一个实例，把它们合并起来

```java
LocalDateTime localDateTime=LocalDateTime.of(date, time)
```

- valueOf:比from和of更繁琐的一种替代方法

```java
String str=String.valueOf(false);
```

- instance/getInstance：返回的实例是通过方法的(如有)参数来描述的，但是不能说与参数具有同样的值


- create/newInstance:与instance/getInstance一样，但其命名确保`每次调用都返回一个新的实例`

```java
Object newArray=Array.newInstance(classObject,arrayLen);
```

- getType:与getInstance一样，但是在工厂方法处于不同的类中的时候使用，Type表示工厂方法所返回的对象类型

```java
FileStore fs = Files.getFileStore(path);
```

- newType:与newInstance一样，但是在工厂方法处于不同的类中的时候使用，Type表示工厂方法所返回的对象类型

```java
 BufferedReader bufferReader=Files.newBufferedReader(Path path)
```

- type:getType/newType简版

```java
List<T> list=Collections.list(t);
```

