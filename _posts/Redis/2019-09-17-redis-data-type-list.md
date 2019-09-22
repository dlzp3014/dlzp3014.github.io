---
layout: post
title:  "Redis数据类型-列表(List)"
date:   2019-09-17 08:38:00
categories: Redis 
tags: Redis-DataType
---

* content
{:toc}

Redis的列表(List)数据类可以被看到简单的字符串列表，列表按照`插入的顺序排序`，在操作Redis列表时，可以将一个元素插入到这个列表的头部或者尾部。一个列表大约可以存储2^32-1个元素





## 向列表中插入值

### LPUSH:将多个值插入列表头部

```java
LPUSH key value [value...]
```

`LPUSH命令用于将一个或者多个value值插入列表key的头部`。如果同时插入多个value值，那么各个value将会按照从左到右的顺序依次插入头部。如对于空列表list，执行命令`LPUSH list a b c`，列表key的值将是c b a [`这个列表相当于一个栈`]

- 当列表key不存在时，将会创建一个空列表，然后执行LPUSH命令

- 如果key存在，但是它不是列表类型的，则执行LPUSH命令将会报错

- 执行LPUSH命令后，返回列表的长度

### RPUSH:将多个值插入列表尾部

```java
RPUSH key value [value...]
```
`RPUSH命令用于将多个或者多个value值插入列表key的尾部`。如果同时插入多个value值，那么各个value值将会按照从左到右的顺序依次插入表尾。如对于空列表list，执行`RPUSH a b c`命令之后，列表的值将是a b c


### LINSERT：插入一个值到列表中

```java
LINSERT key BEFFORE | AFTER pivot value
```

LINSERT命令用于向列表中插入一个值，也就是将值value插入列表key当中，这个值的位置在值pivot之前或者之后。

- 在列表key中，当pivot这个值不存在时，执行该命令无效

- 当列表key不存在时，key将被看作空列表，执行该命令无效

-当key不是列表类型时，将返回一个错误


### LPUSHX:将值插入列表头部

```java
LPUSHX key value
```
LPUSHX命令用于将value值插入列表key的头部，此时key必须存在，且是列表类型。LPUSHX与LPUSH命令相反，当key不存在时，LPUSHX命令不会创建一个新的空列表

### RPUSHX:将值插入列表尾部

```java
RPUSHX key value
```

RPUSHX命令用于当且仅当key存在并且是列表类型时，将value值插入列表key的表尾。RPUSHX与RPUSH命令相反，当key不存在时也不会创建一个新的空列表


### LSET:修改列表元素值

```java
LSET key index value
```

LSET命名用于设置下标为index的列表key的值为value。当下标index参数超出返回或者列表key为空时，将会返回错误。


## 获取列表元素

### LLEN:统计列表的长度

```java
LLEN key
```
LLEN命令用于统计列表key的长度。当key不存在时，key将被视为空列表，返回0

### LINDEX:获取列表元素的值

```java
LINDEX index
```

LINDEX命令用于获取列表key中下标为index的元素

- index参数以0表示列表中的第一个元素，以1表示列表中的第二个元素

- index参数可以为负数，为-1时表示列表中的最后一个元素，为-2时表示列表中的倒数第二个元素

- 当列表存在时，执行该命令后，返回列表中下标为index的元素；当index参数在不列表的范围之内(大于或者小于列表范围)时，执行命令后返回nil


### LRANGE：获取列表指定区间内的元素

```java
LRANGE key start end
```

LRANGE命令用于获取列表key指定区间内的元素，区间从start开始，到end结束。参数start和end都以0为底，即0表示列表中的第一个元素；也可以是负数，即-1表示列表中的最后一个元素

- 当参数start和end的值超出列表的下标值时，不会引起错误

- 当参数stat的值大于列表的最大下标end值时，执行LRANGE命令会返回一个空列表

- 当设定的参数值比下标end值还要大时，Redis将会把这个设定的参数作为列表的end值(最大值)

查询列表中的所有数据可使用`LRANGE key 0 -1`


## 删除列表元素

### LPOP：返回并删除列表的头元素

```java
LPOP key
```

LPOP命令用于返回列表key的头元素，同时把这个头元素删除。key不存在时将返回nil

### RPOP：返回并删除列表的为元素

```java
RPOP key
```
RPOP命令用于返回列表key的尾元素，并把整个尾元素删除

### BLPOP:在指定时间内删除列表的头元素

```java
BLPOP key [key...] timeout
```
BLPOP命令是列表的阻塞式弹出原语，是LPOP的阻塞版本。当列表中没有任何元素key被弹出时，连接将被命令BLPOP阻塞，直到等待超时或者有可弹出元素为止，当设置多个参数key时，将会按照参数key出现的先后顺序来依次检查各个列表，弹出第一个非空列表的头元素


BLPOP命令存在两种行为：阻塞行为和非阻塞行为

阻塞行为：当所有给定的key都不存在或者包含空列表时，该命令将会阻塞连接，直到等待超时或者在另一个客户端中对给定的key执行RPUSH或者LPUSH命令为止。参数timeout是一个以秒为单位的超时时间值，当tiemout为0时，这个阻塞时间可以无限延长

### BRPOP:在指定时间内删除列表的尾部元素

```java
BRPOP key [key] timeout
```

BRPOP命令是列表key的阻塞式命令。当给定列表内没有任何元素可以返回时，连接将被BRPOP命令阻塞，直到等待超时或者发现可返回的元素为止，时RPOP命令的阻塞版本

当同事给定多个参数key时，将会按照参数key的先后顺序检查各个列表，返回第一个非空列表的尾元素



### LREM:：删除指定个数的元素

```java
LREM key count value
```

用于根据参数count的值，删除列表key中与指定参数value相等的元素：

- count = 0时，表示删除列表key中`所有`与value相等的元素

- count > 0时，表示从列表key的`表头`开始向表尾搜索，删除与value相等的元素，删除的数量为count个

- count < 0时，表示从列表key的`表尾`开始向表头搜索，删除与value相等的元素，删除的数量为count的绝对值个

当列表key存在时，该命令返回被删除的元素数量；当列表key不存在时，该命名始终返回0


### LTREM:在指定区间内修剪列表

```java

LTREM key start stop
```

用于对一个列表进行trim操作，如去除不必要的空格，让列表key`只保留指定区间内的元素`，不在这个区间内的元素将会被删除。如执行命令`LTREM list 0 5`，表示只保留列表list中的前6个元素，其余元素将被删除

- 参数start和stop默认值为0，表示列表中的第一个元素;也可以是负数，用-1表示列表中的最后一个元素

- 当参数start和stop值超出列表的下标值时，不会引起错误

- `如果参数start的值比列表下标的最大值还要大，或者start大于stop时，将会清空这个列表`，返回一个空列表

- `如果参数stop的值比列表下标的最大值还要大，redis会将stop的值作为这个列表下标的最大值`


## 移动列表

### RPOPLPUSH：将列表元素移动到另一个列表中

```java
RPOPLPUSH：source destination
```

RPOPLPUSH命令在一个原子时间内，会将列表source中的`最后一个元素弹出，并返回给客户端，这个被返回的元素将会插入列表destination中，作为该列表的头元素`。如果列表source与列表destination相同，那么列表的尾元素将被移动到表头，并返回该元素，这种情况就是列表的旋转操作


### BRPOPLPUSH:在指定时间内移动列表元素到另一个列表中

```java
BRPOPLPUSH source destination timeout
```

BRPOPLPUSH是RPOPLPUSH命令的阻塞版本。当列表source不存在(为空)时，BRPOPLPUSH命令将阻塞连接，直到等待超时，或者被另一个客户端对列表source执行RPUSH或者LPUSH命令为止；当列表source不为空时，BRPOPLPUSH命令执行的效果和RPOPLPUSH命令执行的效果一样

参数timeout表示超时时间，单位为妙，当timeout值为0时，表示阻塞时间可以无限期延长



## 列表模式

- 安全队列

Redis列表经常被看作一个队列，用于在不同程序之间有序交换消息。一个客户端通过LPUSH命令将一条消息放入这个队列中，然后开启另一个客户端通过RPOP或者BRPOP命令取出这个队列中等待时间最长的消息。但是`这个队列是不安全的，当一个客户端在取出一条消息之后遇到错误或者其他原因导致客户端崩溃时，还没有处理完的消息就会丢失`

为了保证未处理完的消息不再丢失，可以使用命令RPOPLPUSH来解决。`RPOPLPUSH命令在返回一条消息的同时会将这条消息保存到另一个列表中，这样就保证了没有处理完的消息不会丢失。在处理完这条消息之后，可以使用LREM从这个备份列表中将其删除`。可以添加一个客户端来监视这个备份列表，这个客户端会自动将超过一定处理时间的消息备份到这个列表中，依次来保证消息不会丢失

- 循环列表

使用RPOPLPUSH命令可以实现循环列表。使用相同的key作为RPOPLPUSH命令的两个参数，客户端采用逐个获取列表元素的方式，取出列表中的所有元素。这样就避免了像使用LRANG命令那样同时取出列表中的所有元素

 - 当有多个客户端同时对一个列表进行旋转操作来获取不同元素，直到所有元素被取出完，之后又开始循环时，这个循环队列可以正常工作

 - 当有客户端向列表尾部添加新元素时，这个循环列表也能正常工厂

 基于以上两种情况，`可以借助Redis实现服务器监控系统，在短时间内连续不断地处理一些消息`


循环队列模式是安全的，它易于扩展，当遇到处理消息的客户端突然崩溃的情况下，消息也不会因此而丢失，等下一次循环时，其他客户端也能处理这些消息


















