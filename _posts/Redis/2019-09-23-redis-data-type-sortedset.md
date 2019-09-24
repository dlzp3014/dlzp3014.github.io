---
layout: post
title:  "Redis数据类型-有序集合(Sorted Set)"
date:   2019-09-23 08:38:00
categories: Redis 
tags: Redis-DataType
---

* content
{:toc}

Redis的数据类型有序集合(Sorted Set)也是String类型的集合。有序集合中`不存在重复的元素`，每个集合元素都有一个对应的`double类型的分数`。Redis就是通过这个元素对应的分数来为集合元素进行从小到大的排序的。`集合中的元素是唯一的，但是集合元素所对应的分数值不唯一，可以重复`


有序集合采用哈希表实现，当面对增加、删除、查询操作时，效率特别高，复杂度为O(1)。有序集合中所能存储的最大元素数量是2^32-1个





## 有序集合添加元素

### ZADD:添加多个元素到有序集合中

```java
ZADD key score member [[score member] ...]
```

用于将一个或者多个member元素及它对于的`score(分数)值`加入到有序集合key中。如果`有序集合key中已经存在某个member元素，那只更新这个member元素的source值，然后重新插入member元素，以此来确保member元素在正确的位置上`。score值可以是double类型的浮点数，也可以是整数值。如果有序集合key不存在，则创建一个新的有序集合，然后执行ZADD操作。`ZADD执行成功后将会返回添加的新元素的数量，不包括那些被更新的，已经存在的元素`



### ZINCRBY：为分数值加上增量

```java
ZINCYBY key increment member
```

用于为有序集合key中的member元素的score值加上增量increment。当increment是一个负数时，表示让score减去相应的值。当key不存在或者有序集合key没有member元素时，ZINCRBY key increment member命令等价于ZADD key increment

参数score值可以是double类型的浮点数，也可以是整数值(正负数)。ZINCRBY命令执行成功后，会以字符串的形式返回有序集合key中的member元素的score值


## 获取有序集合元素

### ZCARD:获取有序集合中的元素数量

```java
ZCARD key
```

当key存在且是有序集合时，返回有序集合中的元素个数。当key不存在时，返回0


### ZCOUNT:获取在分数区间内的元素数量

```java
ZCOUNT key min max
```

ZCOUNT命令用于获取有序集合key中，score值在min和max之间(`默认包含score值等于min或max`)的元素数量

### ZLEXCOUNT：获取在指定区间内的元素数量

```java

ZLEXCOUNT key minMember maxMember
```

ZLEXCOUNT命令用于获取有序集合key中介于min和max范围内的元素数量，这个`有序集合key中的所有元素的score值都是相等`

参数min和max是一个区间，区间一般使用'('或者'['表示。其中，'('表示开区间，指定的值不会被包含在范围之内；'['表示闭区间
，指定的值会被包含在范围之内。特殊值+和-在参数min和max中具有特殊含义，其中+表示正无穷，-表示负无穷。如向一个元素分数相同的有效集合发生命令ZLEXCOUNT <zset> -+将返回这个有序集合中的所有元素

example:
```java


ZLEXCOUNT sorted:set - + 获取有序集合的所有元素数量 -> countMember

ZLEXCOUNT sorted:set (memb1 [memb2

```


### ZRANGE：获取在指定区间内的元素(升序)

```java
ZRANG key start stop [WITHSCORES]
```

ZRANGE命令用于返回有序集合key中指定区间内的元素，返回的元素按照score值从小到大的顺序排序。具有相同score值的元素会按照字典序排序。参数start stop可以为负数，0表示有序集合中的第一个元素，-1表示有序集合中的最后一个元素，超出有序集合中的下标不会引起错误

- 当start的值大于有序集合key的最大下标或者start大于stop时，ZRANGE命令指示简单地返回一个空列表

- 当stop的值大于有序集合key的最大下标时，Redis会将这个stop的值作为有序集合key的新下标

- WITHSCORES选项用于实现同时返回集合元素和这些元素所对应的score值，格式为:value,score ...，返回的元素的数据类型可能会很复杂，如元组、数组等

```java

ZRANG sorted:set 0 -1 

ZRANG sorted:set 0 -1 WITHSCORES

```


### ZREVRANGE：获取在指定区间内的元素(降序)

```java
ZREVRANGE key start stop [WITHSCORES]
```

ZREVRANGE与ZRANGE命令相反，返回的元素按照score值从大到小顺序排序，相同的socre值的元素，按照字典序的逆序排序

### ZSCORE:获取元素的分数值

```java
ZSCORE key member
```

用于返回有序集合key中member元素的score值，命令执行成功后，以字符串的形式返回member元素的score值

### ZRANGEBYLEX:获取集合在指定范围内的元素

```java
ZRANGEBYLEX key min max [LIMIT offset count]

```
用于返回有序集合key中元素score值介于min和max之间的元素，这个有序集合key中的所有元素具有`相同的score值`，它们安装字典顺序排序。`如果有序集合key中的元素对应的score值不同，则在执行该命令后，返回的结果时未指定的(unspecified)`

可选的LIMIT offset count参数用于获取指定范围内的匹配元素。如果offset参数的值非常大，那么该命令在返回结果之前，需要先遍历到offset所指定的位置。

参数min和max是一个区间，具体说明参考[ZLEXCOUNT]命令

```java

ZRANGEBYLEX sorted:equle:set - + 全部元素

ZRANGEBYLEX sorted:equle:set - [member 小于等于member的所有元素

```

### ZRANGEBYSCORE:获取在指定分数区间内的元素

```java
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```

用于返回有序集合key中，所有score值介于min和max之间(包含等于min和max)的元素。有序集合key中的元素按照score值从小到大的顺序排序。`当不知道min和max参数的具体指示，可以使用-inf来表示min值，使用+inf来表示max值`。在默认情况下，min和max区间时闭区间，也可是在参数前面添加'('符号来使用可选的开区间。

- 当具有相同score值的元素时，有序集合元素会按照字典排序

- 使用WITHSCORES选项来返回元素的score值

- 可选的LIMIT offset count参数用于获取指定范围内的匹配元素，如果offset参数的值非常大，那么该命令在返回结果之前，需要先遍历到offset所指定的位置

```java
ZRANGEBYSCORE sorted:set score01 score02 WITHSCORES

ZRANGEBYSCORE sorted:set -inf score02 WITHSCORES

```

### ZREVRANGEBYSCORE:获取在指定分数区间内的元素

```java
ZREVRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```

与ZRANGEBYSCORE命令功能相同，返回的元素按照score值从大到小的顺序排序



## 有序集合排名

### ZRANK:获取有序集合元素的排名

```java
ZRANK key member
```

用于获取有序集合key中member元素的排名。其中有序集合元素会按照score值从小到大的顺序排序，排序以0为底，也就是score值最小的元素排名为0


### ZREVRANK:获取有序集合元素的倒序排名

```java
ZREVRANK key member
```

有序集合元素按照score值从大到小的顺序排序，排名以0为底，也就是score值最大的成员排名为0


## 有序集合运算


### ZINTERSTORE:保存多个有序集合的交集

```java
ZINTERSTORE destination numkeys key [key...] [WEIGHTS weight [weight...]] [AGGREGATE SUM|MIN|MAX]
```

用于计算给定的一个或者多个有序集合key的交集，其中给定key的数量必须和numkes相等，并将该交集存储到destination中。默认情况下，交集(结果集)中的某个元素的score值时所有给定有序集合中该元素的score值之和。命令执行成功后，返回保存到destination结果集中的元素个数


### ZUNIONSTORE：保存多个有序集合的并集

```java
ZUNIONSTORE destination numkeys key [key...] [WEIGHTS weight [weight...]] [AGGREGATE SUM|MIN|MAX]

```

用于计算给定的一个或多个有序集合key的并集，给定key的数量必须等于numkeys，并将计算的并集结果保存到destination中，默认情况下，并集结果中的元素的score值是所有指定结合中该元素的score值值和

使用WEIGHTS选项来为每个给定的有序集合分别指定一个乘数，每个给定的有序集合中的所有元素的score值在传递给聚合函数之前都会乘以这个乘数(weight)。如果没有指定WEIGHTS选项，则乘数默认设置为1


使用AGGREGATE选项来指定计算并集结果的聚合方式：

- SUM:默认的聚合方式，将所有有序集合中某个元素的score值之和作为结果集中该元素的score值

- MIN:将所有有序集合中某个元素的最小score值作为结果集中该元素的score值

- MAX:将所有有序集合中某个元素的最大score值作为结果集中该元素的score值


## 删除有序集合

### ZREM：删除有序集合中的多个元素

```java
ZREM key member [member ...]
```

删除有序集合key中的一个或多个元素，不存在的元素会被忽略，命令执行后返回被删除元素的数量，不包括被忽略的元素


### ZREMRANGEBYLEX:删除有序集合在指定区间内的元素

```java
ZREMRANGEBYLEX key memb1 memb2
```

删除有序集合key中`介于min和max范围的score值相同的所有元素`。参数min和max与ZRANGEBYLEX命令的意义相同

```java

ZREMRANGEBYLEX sorted:set - + 删除有序集合中的所有元素


ZREMRANGEBYLEX sorted:set  (member +

```

### ZREMRANGEBYRANK:删除有序集合在指定排名区间内的元素

```java
ZREMRANGEBYRANK key start stop
```

删除有序集合key在指定排名(rank)区间内的所有元素，区间范围有下标参数start和stop给出，用法与ZRANGE中下标参数start和stop的用法相同。命令执行成功后返回被删除元素的数量

### ZREMRANGEBYSCORE:删除有序集合在指定分数区间内的元素

```java
ZREMRANGEBYSCORE key min max
```

用于删除有序集合key中，所有score值介于min和max之间(包含等于min或max)的元素，命令执行成功后，返回被删除元素的数量










