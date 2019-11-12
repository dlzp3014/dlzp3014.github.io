---
layout: post
title:  "Redis数据类型-字符串(String)"
date:   2019-09-25 08:38:00
categories: Redis 
tags: Redis-DataType
---

* content
{:toc}


字符串类型时Redis中最基本的数据类型，是二进制安全的，任何形式的字符串都可以存储，包括二进制数据、序列化后的数据、json化的对象，甚至是BASE64编码后的图片。String类型的键最大能存储512MB的数据.具体操作如下：




## 设置键值对

### SET:设置键值对

```java
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```

使用SET命令将字符串值value设置到key中。如果key已经存在其他键，则在执行SET命令后，将会覆盖旧值，并且忽略类型。`针对某些带有生存时间的key来说，当SET命令成功执行时，key上的生存时间会被清除`

SET命令的可选参数如下：

- EX seconds: 用于设置key的过期时间为多少秒(seconds)。SET key value EX seconds等价于SETEX key seconds value

- PX milliseconds: 用于设置key的过期时间为多少毫秒(milliseconds value)。SET key value PX milliseconds等价于PSETEX key milliseconds value

- NX: 表示key不存在时，才对key进行设置操作。SET key value NX 等价于 SETNX key value

- XX: 表示当key存在时，才对key进行设置操作

如果SET命令设置成功，则会返回OK。如果设置了NX或者XX，但因为条件不足而设置失败，则会返回空批量回复(NULL Bulk Reply)


### MSET:设置多个键值对

```java
MSET key value [key value...]
```

使用MSET命令时可以设置多个键值对。如果某个key已经存在，则会用新值覆盖旧值。`MSET命令是一个原子性操作，所有给定key都会在同一时间内被设置更新，不存在某些key被更新了而另一些key没有被更新的情况`


### SETNX:设置不存在的键值对


```java
SETNX key value
```

SETNX是set if not exists的缩写。`当且仅当key不存在时，才进行设置，如果已经存在，则什么也不做`。SETNX命令设置成功返回1，设置失败返回0，如下返回值：

```java

> SET session:id xxxxx
1

> SET session:id xxxxx
0

```

### MSETNX:设置多个不存在的键值对

```java
MSETNX key value [key value...]
```

`当且仅当所有给定key都不存在时才设置成功，如果有一个给定的key已经存在，那么MSETNX命令也会拒绝执行所有给定key的设置操作。MSETNX命令是原子性的，可以同来设置多个不同key表示不同字段的唯一性逻辑对象，所有字段要么全部被设置，要么全部设置失败`



## 获取键值对

### GET:获取键值对的值

```java
GET key
```

用于获取key中设置的字符串值(GET命令只能用于处理字符串的值)


### MGET:获取多个键值对的值

```java
MEGT key [key...]
```

MGET命令同时返回多个给定key的值，key之间是有空格隔开。MGET命令执行后返回一个包含所有给定key的值的列表

### GETRANGE:获取键的字符串值

```java

GETRANGE key start end
```

用来获取key中字符串值从start到end结束的子字符串。下标从0开始(字符串截取)。start和end参数是整数，可以取负值，当取负值时，表示从字符串最后开始计数，-1表示最后一个字符。



## 键值对的偏移量

### SETBIT:设置键的偏移量

```java
SETBIT key offset value
```

用于对key所存储的字符串设置或者清除指定偏移量上的位(bit)，value参数值决定了位的设置或清除，值取0或1。当key不存在时，自动生成一个新的字符串值。这个字符串是动态的，可以扩展以确保将value保持到指定的偏移量上。当这个字符串扩展时，使用0来填充空白位置。offset参数必须是大于或等于0，且小于2^31(bit映射被限制在512M之内)的正整数。默认情况下，bit初始化为0。SETBI命令返回指定偏移量原来存储的位


### GETBIT:获取键的偏移量值

```java
GETBIT key offset
```

对key所存储的字符串值，使用GETBIT命令来获取指定偏移量上的位(bit)。当offset的值超过字符串的最大长度或者key不存在时，返回0


## 设置键的生存时间

### SETEX:为键设置生存时间(秒)

```java
SETEX key seconds value
```

将value值设置到key中，并设置key的生存时间为多少秒(seconds)，如果key已经存在，则将覆盖旧值。等价于如下命令：

```java
SET key value
EXPIRE key seconds
```

SETEX命令具体一个原子性，它这设置value与设置生存时间是在同一时间完成


### PSETEX:为键设置生存时间(毫秒)

```java
PSETEX key milliseconds value
```

设置key的生存时间，以毫秒为单位


## 键值对的值操作

### SETRANGE:替换键的值

```java
SETRANGE key offset value
```

从指定的位置(offset)开始将key的值替换为新的字符串，如果有不存在的key，就当做空白字符串处理。SETRANGE明辉会确保字符串足够长，以便将value设置在指定的偏移量上。如果给定key原来存储的字符串长度比偏移量小(如：字符串只有4个字符，但设置的offset是9)，那么原字符串和偏移量之间的空白将用零字节(Zerobytes:'\x00')来填充。SERRANGE命令之后之后返回字符串的长度

### GETSET：为键设置新值

```java
GETSET key value
```

将给定key的值设置为value，并返回key的旧值。当key存在但不是字符串类型时，将返回错误

```java
>EXISTS msg:key
0 # 无对应的键

> GETSET msg:key 'hello'
> # 未返回值

> GET msg:key
> hello

> GETSET msg:key 'hello word'
> hello

> GET msg:key

```

### APPEND:为键缀加值

```java
APPEND key value
```

将value值追加到key旧值，如果key不存在，则将key设置为value


## 键值对的计算

### BITCOUNT:计算比特位数量

```java
BITCOUNT key [start] [end]
```

计算给定的字符串中被设置为1的比特位数量，如果不设置这两个参数，则表示会对整个字符串进行计算；如果设置了这两个参数值，则可以让计数只在特定的位上进行。start和end参数都是整数值，可以取负数


### BITTOP:对键进行位运算

```java
BITTOP operation destkey key [key...]
```

对一个或多个保存二进制位的字符串key进行位运算，并将运算结果保存到destkey中。operation表示为操作符，可以是AND、OR、NOT、XOR这4种操作中的任意一种，具体操作如下：

- BITTOP AND destkey key [key...]: 表示对一个或多个key求逻辑并，并将结果保存到destkey中

- BITTOP OR destkey key [key...]: 表示对一个或多个key求逻辑或，并将结果保存到destkey中

- BITTOP NOT destkey key : 表示对一个或多个key求逻辑非，并将结果保存到destkey中

- BITTOP XOR destkey key [key...]: 表示对一个或多个key求逻辑异或，并将结果保存到destkey中

当使用BITTOP命令来进行不同长度的字符串的位运算时，较短的那个字符串所缺少的部分将被看作0，空的key也被看作包含0的字符串序列


### STRLEN:统计键的值的字符长度

```java
STRLEN key
```

获取key的值的字符长度，当key不存在时返回0

## 键值对的值增量

### DECR:键的值减1

```java
DECR key
```

将key中存储的数值值减1，如果key不存在，则key的值先被初始化为0，再执行DECR操作减1，这个命令只能对数字类型的数据进行操作(自减)。DECR操作的值限制在64位有符号数字表示范围之内


### DECRBY:键的值减去减量值

```java
DECRBY key decrement
```

将key所存储的值减去减量值decrement，如果key不存在，则key的值先被初始化为0，再执行DECRBY命令。


### INCR:键的值加1

```java
INCR key
```

将key中存储的数字值加1，如果key不存在，则key的值先被初始化为0，在执行INCR操作加1

### INCYBY:键的值加上增量值

```java
INCYBY key increment
```

将key所存储的值加上增量值increment，如果key不存在，则key的值先被初始化为0，再执行INCRBY

### INCRBYFLOAT:键的值加上浮点数增量值

```java
INCRBYFLOAT key increment
```

将key所存储的值加上浮点数增量值increment，如果key不存在，则key的值先被初始化为0，再执行INCRBYFLOAT，产生的新值和浮点数增量值increment都可以使用4e7这样的指数符号来表示。执行INCRBYFLOAT命令之后，产生的值会以相同的形式存储，都是一个数字、一个可选的小数点、一个任意位数的小数部分组成，小数部分末尾的0会被忽略掉。在有特定需要时，浮点数也会被转换为整数。

在INCRBYFLOAT命令之后，无论产生的浮点数的实际精度有多长，计算结果最多也只能保留小数点后17位















