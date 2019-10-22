---
layout: post
title:  "Redis命令-排序(STOT)命令"
date:   2019-10-10 08:38:00
categories: Redis 
tags: Redis-Command
---

* content
{:toc}

Redis的SORT命令用于对相关数据进行排序，具体可以对`列表键、集合键或有序集合键的值`进行排序，其用法如下：



## SORT排序命令

- 使用SORT命令实现列表键的排序：

```java

127.0.0.1:6379> RPUSH student:numbers 2 1 3 5 4
(integer) 5

127.0.0.1:6379> LRANGE student:numbers 0 -1
1) "2"
2) "1"
3) "3"
4) "5"
5) "4"

127.0.0.1:6379> SORT student:numbers # 对编号进行排序
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"

```

`SORT <key>` 是SORT命令的最简单形式，用于实现对列表key的排序，这个列表key包含数字值


- 使用SORT命令对有序集合进行排序:

【Note:】 `会忽略有序集合元素的分数，而只对元素的值进行排序`


```java
127.0.0.1:6379> ZADD student:score 96 2 95 1 98 3 94 4
(integer) 4

127.0.0.1:6379> ZRANGE student:score 0 -1 //根据分数从小到大排序，展示value
1) "4"
2) "1"
3) "2"
4) "3"

127.0.0.1:6379> SORT student:score  //对有序集合的value进行排序，与score没有关系
1) "1"
2) "2"
3) "3"
4) "4"
```

- 使用ALPHA参数，对字符串值的键进行排序(`SORT <key> ALPHA`):

```java
127.0.0.1:6379> LPUSH color:list red black blue green
(integer) 4

127.0.0.1:6379> LRANGE color:list 0 -1
1) "green"
2) "blue"
3) "black"
4) "red"

127.0.0.1:6379> SORT color:list ALPHA
1) "black"
2) "blue"
3) "green"
4) "red"

```

在没有为SORT命令设置ALPHA参数的条件下，如果使用SORT命令对字符串值进行排序，将会出现错误信息`(error) ERR One or more scores can't be converted into double`

SORT命令会尝试将所有元素转为双精度浮点数来进行比较，如果转换错误就会报错



## ASC与DESC 参数

在默认情况下，使用SORT命令排序后，排序结果将会按照从小到大的顺序排列。在实际应用中，常常需要对一些数据进行降序(从大到小)排列，此时可以为SORT命令设置DESC参数，让排序结果降序排列。在使用SORT命令实现从小到大的排序过程中，常常会省略ASC参数，其`SORT <key>命令等价于SORT <key> ASC命令`。为SORT命令设置ASC或者DESC参数实现排序的操作如下：

```java
127.0.0.1:6379> sort color:list ASC ALPHA //ASC 升序
1) "black"
2) "blue"
3) "green"
4) "red"

127.0.0.1:6379> SORT color:list DESC ALPHA //DESC 降序
1) "red"
2) "green"
3) "blue"
4) "black"

```

`Redis的升序和降序都是由快速排序算法实现`

## BY 参数

在默认情况下，使用SORT命令进行排序会按照元素本身的值进行排序，元素本身决定了元素在排序之后所处的位置。如使用ALPHA参数对字符串集合进行排序时，是按照元素在字典中的顺序得出的。如果想`按照其他键来排序，则可以通过BY参数来实现`

使用BY参数时，SORT命令可以指定某些字符串键或者某个哈希键所具有的某些域来作为排序依据，对这个件进行排序。如下采用颜色的RGB值(256,256,256)对颜色进行排序：


```java
127.0.0.1:6379> SADD color:set gree blue red
(integer) 3

127.0.0.1:6379> MSET gree-RGB 91 blue-RGB 234 red-RGB 80
OK

127.0.0.1:6379> SORT color:set BY *-RGB
1) "red"
2) "gree"
3) "blue"
```

服务器执行SORT color:set BY \*-RGB命令执行过程：

- 服务器接收到命令之后，进行解析，会根据命令创建一个redisSortObject结果的数组，数组的长度就是color集合的大小

- 遍历这个数组，将每个数组元素的obj指针分别指向color:set集合中的每一个元素，然后根据每个数组元素的obj指针所指向的集合元素，以及BY参数所指定的字符串键\*-RGB，查找相对应的权重键。如green元素对应的权重键就是green-RGB

- 服务器会将这些元素的权重键所对应的权重值转化为双精度浮点数，然后保存到响应数组向的u.score属性中。如green元素的权重键green-RGB的值转化为浮点数后为"91.0"

- 以数组向的u.score属性的值为`权重`，按照从小到大的顺序对数组进行排序，将会得到一个升序的数组

- 遍历这个新数组，依次将数组项的obj指针所指向的集合元素返回给客户端


在默认情况下，BY参数排序的权重键保存的值为数字值。`如果这些权重键中保存的值是字符串值，那么要实现对这些权重键的排序，除使用BY参数之外，还需要使用ALPHA参数`，命令格式如下：

```java
SORT <key> BY <pattern> ALPHA
```

当将一个不存在的键作为参数传递给BY参数时，SORT命令将跳过排序操作，直接返回结果。因此，在实际应用中，需要根据实际情况选择`合适的参数相结合`进行排序


## LIMIT 参数

在使用SORT命令进行排序时，不管有多少个元素，安排后都会返回所有的元素到客户端。如果使用SORT命令对一个很大的集合进行排序，同时又不希望SORT命令返回这个集合排序结果的所有元素，而只需要其中的`一部分元素时，则可以使用SORT命令的LIMIT参数来实现`，命令格式如下：

```java
LIMIT <offset> <count>
```

- offset参数：要跳过的已排序元素数量

- count参数：在跳过offset个已排序的元素之后，要返回多少个已排序的元素

```java
127.0.0.1:6379> SORT color:set ALPHA LIMIT 0 -1
1) "blue"
2) "gree"
3) "red"
127.0.0.1:6379> SORT color:set ALPHA LIMIT 1 2
1) "gree"
2) "red"
```


## GET与STROE 参数

- GET参数


使用SORT命令对键进行排序后，在默认情况下，总是返回被排序键本身所包含的元素，如果想要得到这些排序键所对应的值，则可以使用GET参数。GET参数不参与排序，其作用是使用SORT命令的返回结果不再是元素本身的值，而是GET参数所指定的默认匹配的值。GET参数和BY参数一样，也支持字符串类型和散列类型的键，并使用"\*"作为模式匹配符。如果根据城市的简写排序后获取全名称，操作如下：

```java
127.0.0.1:6379> SADD citys xian shenzhen chengdu
(integer) 3

127.0.0.1:6379> SORT citys ALPHA
1) "chengdu"
2) "shenzhen"
3) "xian"

127.0.0.1:6379> MSET  xian:name shanxi-xian shenzhen:name guangdong-shenzhen chengdu:name sichuan-chengdu
OK

127.0.0.1:6379> SORT citys ALPHA GET *:name

1) "sichuan-chengdu"
2) "guangdong-shenzhen"
3) "shanxi-xian"

```

`一个SORT命令可以带多个GET参数，但只能带有一个BY参数。随着GET参数的增多，SORT命令要执行的查找操作也会增多`。如上示例，排序后还要获取GDP信息：

```java
127.0.0.1:6379> MSET xian:GDP 8891 shenzhen:GDP 9812 chengdu:GDP 12323
OK

127.0.0.1:6379> SORT citys ALPHA GET *:name GET *:GDP
1) "sichuan-chengdu"
2) "12323"
3) "guangdong-shenzhen"
4) "9812"
5) "shanxi-xian"
6) "8891"
```

也可以使用`GET #`参数返回包含排序元素本身的值

```java
127.0.0.1:6379> SORT citys ALPHA GET *:name GET *:GDP GET #
1) "sichuan-chengdu"
2) "12323"
3) "chengdu"
4) "guangdong-shenzhen"
5) "9812"
6) "shenzhen"
7) "shanxi-xian"
8) "8891"
9) "xian"
```

除将字符串键作为GET或BY参数之外，还可以使用`哈希表作为GET或BY参数`，如此操作：

```java

127.0.0.1:6379> LPUSH user:id 1 2 //列表存储用户编码
(integer) 2

127.0.0.1:6379> HMSET user:info:1 name dlzp_01 age 32 //散列储存用户信息
OK

127.0.0.1:6379> HMSET user:info:2 name dlzp_02 age 28
OK

127.0.0.1:6379> SORT user:id BY user:info:*->age //按照age排序
1) "2"
2) "1"


127.0.0.1:6379> SORT user:id BY user:info:*->age GET user:info:*->name //按照age排序，获取name
1) "dlzp_02"
2) "dlzp_01"
```


- STORE参数

在默认情况下,SORT命令进行排序，只会向客户端返回排序的记过，并不会保存这些排序结果。如果想把排序结果保存起来，则可以使用STORE参数。`通过使用STORE参数，可以把排序结果保存到指定的键中，在需要的时候从这个键中取出`

```java
127.0.0.1:6379> SORT citys alpha
1) "chengdu"
2) "shenzhen"
3) "xian"

127.0.0.1:6379> SORT citys ALPHA STORE order:citys
(integer) 3

127.0.0.1:6379> LRANGE order:citys 0 -1
1) "chengdu"
2) "shenzhen"
3) "xian"
```

`STORE参数保存的键时列表类型的，如果这个键已经存在，则新的键会覆盖旧键(情况里面的元素，然后保存排序后的元素)。SORT命令添加上STORE命令后，其返回结果为结果的个数`


## SORT多参数执行顺序

在使用Redis的SORT命令进行排序时，一般都会携带多个参数，而这些参数的执行顺序是有先后的。按照参数的执行顺序，SORT命令的执行过程可以划分为以下几步：

- 进行排序：在这一步中，可以使用ALPHA参数、ASC或者DESC参数及BY参数实现排序，并返回结果

- 对排序结果集的长度限制：在这一步中，可以使用LIMIT参数对排序结果进行限制，返回部分排序结果，并保存到排序结果集中

- 获取外部键：在这一步中，可以使用GET参数，根据排序结果集中的元素及GET参数指定的模式进行匹配，查找符合要求的键值，同时将这些键值作为新的排序结果集

- 将排序结果集返回给客户端：在这一步中，排序结果集被遍历，并返回给客户端

SORT命令的这4步执行过程环环紧扣，只有当前一步完成之后，才会执行下一步。执行过程命令解析如下：

```java
SORT <key> 
ALPHA ASC|DESC BY <by-pattern>  //排序
LIMIT <offset> <count> //限制长度
GET <get-pattern> //获取外部键
STORE <store:key>  //存储
```


`在使用SORT命令携带多个参数进行排序时，除GET参数之外，其他参数的摆放顺序并不会影响到排序结果`。如果排序命令中含有多个GET参数，必须要保证GET参数的顺序，才能得到想要的排序结果
