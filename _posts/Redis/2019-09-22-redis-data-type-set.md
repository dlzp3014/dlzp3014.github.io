---
layout: post
title:  "Redis数据类型-集合(Set)"
date:   2019-09-22 08:38:00
categories: Redis 
tags: Redis-DataType
---

* content
{:toc}

Redis的数据类型集合Set是String类型的无序集合。集合无序且不存在重复的元素，每个原始都是唯一的。集合时通过哈希表来实现的，所有使用集合进行增加、删除、查询操作时的效率特别高，复杂度为0(1)。一个集合所能存储的最大容量为2^32-1个元素




## 向集合中添加元素

### SADD:添加多个元素到集合中

```JAVA
SADD key rember [rember ...]
```

SADD命令用于将一个或者多个member元素添加到集合key中。如果这个集合key中已经存在这个member元素，那么它将被忽略。如果集合key不存在就创建一个集合，这个集合中只包含这里设置的member元素。命令执行成功后返回被添加到集合中的新元素的数量，不包含忽略的元素

### SMOVE:移动集合元素到另一个集合中

```java
SMOVE source destination member
```

用于将集合source中的member元素移动到集合destination中。SMOVE命令是原子性操作，那么执行成功，要么不执行。如果集合source不存在，或者集合source中不村子member元素，则SMOVE命令不执行任何操作，将返回0；如果集合source中包含member元素，那么SMOVE命令会`将member元素从集合source移动到集合destination中`。

### SUNIONSTORE：保存多个集合元素到新集合中


```java
SUNIONSTORE destination key [key ...]

```

用于获取一个或多个集合key中的全部元素，并将这些元素保存到集合destination中，这个`集合中的元素是给定的集合key元素的交集`。`当只有一个集合key时，执行该命令后，产生的集合destination就是这个集合key本身`。该命令执行后，返回这个交集集合destination中的元素数量


## 获取集合元素

### SISMEMBER:判断某个元素是否在集合中

```java
SISMEMBER key number
```

SISMEMBER命令用于判断元素number是否在集合key中，即就是判断这个元素number是否是集合key的成员。`如果集合key中存在元素number，则返回1；如何集合key中不存在元素number或者集合key不存在，就返回0`


### SCARD：获取集合中元素的数量

```java
SCARD key
```

用于获取集合key中的元素的数量，当集合key不存在时，返回0


### SMEMBERS:获取集合中的所有元素

```java
SMEMBERS key
```

用于获取集合key中的所有元素，如果集合key不存在，则会被看作空集合




### SRANDMEMBER:随机获取集合中的一个元素

```java
SRANDMEMBER key [count]
```

用于随机返回集合key中的一个元素，当且仅当只有参数key时。在后续版本中，添加了参数count。参数count可以是一个正数也可以是一个负数

- 当count为正数且小于集合基数(集合元素个数的最大值)时，执行该命令后返回一个包含count个元素的数组，数组中的元素个各相同。当count大于等于集合基数时，返回整个集合

- 当count为负数时，执行该命令后返回一个元素可能重复多次的数组，整个数组的长度是count的绝对值

SRANDMEMBER与SPOP命令的功能类似，`SPOP在从集合中随机删除元素的同事返回整个元素，而SRANDMEMBER命令只随机返回元素，并不会改动整个集合的内容`

如果集合为空，则返回nil；如果只设置key参数，则将会随机返回一个元素


### SUNION:获取多个集合中的所有元素

```java
SUNION key [key ...]
```

SUNION用于获取一个或多个集合key中的全部元素，这个返回的集合是`所有给定集合key的并集`。如果集合key不存在，则会被看作空集合。`SUNION只是单纯地返回并集元素列表，并不会保存这些元素`。如果多个集合中含有相同的元素，那么SUNION执行后会`忽略重复`元素【Redis的集合元素是无序、不可重复的】


## 集合运算

### SDIFF:获取多个集合元素的差集

```java

SDIFF key [key ...]

```

用于获取一个或者多个集合的全部元素，该集合是所有给定集合之间的差集，集合key不存就视为空集合


### SDIFFSTORE:获取多个集合差集的元素个数

```java

SDIFFSTORE destination key [key ...]
```

用于获取一个或多个集合key的全部元素，并将获取的元素保存到集合destination中，这个集合是给定的多个集合key的元素差集。如果集合destination已经存在，则会被新的元素覆盖；如果给定的集合key是一个而不是多个，那么这个destination集合就是给定的集合key本身。命令执行成功后返回新集合destination中的元素数量


### SINTER:获取多个集合元素的交集

```java
SINTER key [key ...]
```

用于获取给定的一个或者多个集合key中的全部元素，该集合是所有给定集合的交集【返回的交集中的元素并不会被保存】。如果集合key不存在，则会被看作空集合。`如果给定的多个集合key中有一个是空集合，那么该命令执行后的结果就是一个空集合`

### SINTERSTORE:获取多个集合交集的元素个数

```JAVA
SINTERSTORE destination key [key ...]
```

用于获取给定的一个或者多个集合key中的全部元素，并将这些元素保存到集合destination中，而不像SINTER那样只是简单地返回。如果集合destination已经存在，则会被新产生的集合覆盖。`当有一个集合key时，该命令返回这个destination集合就是集合key本身`。SINTERSTORE命令执行后返回集合destination中的成语数量



## 删除集合元素

### SPOP：删除集合中的元素

```java
SPOP key [count]
```

SPOP用于`随机删除`集合key中的一个或多个元素。SPOP命令执行成功后，返回被删除的随机元素。如果集合key不存在或者集合key为空，则返回nil


### SREM：删除集合中的多个元素

```java
SREM: key member [member ...]
```

用于删除集合key中一个或多个member元素。该`命令在执行过程中会忽略不存在的member元素`













