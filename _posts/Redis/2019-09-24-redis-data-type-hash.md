---
layout: post
title:  "Redis数据类型-哈希(Hash)"
date:   2019-09-24 08:38:00
categories: Redis 
tags: Redis-DataType
---

* content
{:toc}

Redis的Hash类型时一个String类型的域(field)和值(value)的映射表，`Hash数据类型常常用来存储对象信息`。在Redis中，每个哈希表可以存储2^32-1个键值对，也就是40多亿个数据





## 设置哈希表域的值

### HSET:为哈希表的域设值

```java
HSET key field value
```

将哈希表key中的filed的值设置为value。当这个key不存在时，将会创建一个新的哈希表进行HSET操作。如果field已经存在于哈希表中，那么新值将会覆盖旧值

### HSETNX:为哈希表不存在的域设值

```java
HSETNX key field value
```

当且仅当field`不存在时`，将哈希表key中的field的值设置为value。如果field已经存在，HSETNX命令将执行无效。如果key不存在，则会首先创建一个新的key，然后执行HSETNX命令。设置成功则返回1，如果field已经存在，设置失败返回0

### HMSET:设置多个域和值到哈希表中

```java
HMSET key field value [field value ...]
```

将一个或多个域-值(field-value)对设置到哈希表key中，该命令执行后，将`覆盖哈希表key中原有的域`。当key不存在时，会创建一个空的哈希表并执行HMSET操作


## 获取哈希表中的域和值

### HGET：获取哈希表中域的值

```java
HGET key field
```

用于获取哈希表key中field值


### HGETALL:获取哈希表中所有的域和值

```java
HGETALL key
```

如果这个key不存在时则返回空列表


### HMGET：获取多个域的值

```java
HMGET key field [field ...]

```

返回一个包含多个指定域(field)的关联值的表，表中的值的顺序与给定域参数的请求顺序保持一致


### HKEYS：获取哈希表中的所有域

```java
HKEYS key
```

返回包含这个哈希表key中的所有域表

### HVALS:获取哈希表中所有域的值

用于返回哈希表key中所有域的值的表


## 哈希表统计

### HLEN:统计哈希表中域的数量

```java
HLEN key
```

返回哈希表key中域的数量，是一个数值。如果key不存在，则返回0，表示一个域也没有

### HSTRLEN:统计域的值的字符串长度

```java
HSTRLEN key field 
```

用于统计哈希表key中域给定域(field)相关联的值的字符串长度。当key或field不存在时，返回0

### HINCRBY:为哈希表中的域加上增量值

```java
HINCRBY key field increment
```

这个increment增量值可以是一个负数，相当于对这个field的值进行减法操作。如果key不存在，则创建新的哈希表key，然后继续执行HINCRBY命令。如果field不存在，则将把field的值初始化为0，然后执行命令。命令执行成功后，返回新值，也就是哈希表key中域(field)的值

`在执行increment命令时，必须保证field时数值类型。HINCRBY命令操作的值被限制在64位有符号数字表示范围之内`


### HINCYBYFLOAT:为哈希表中的域加上浮点数增量值

```java
HINCYBYFLOAT key field increment
```

在Redis中，数字和浮点数都以字符串的形式进行保存。


## 删除哈希表中的域

### HDEL:删除哈希表中的多个域

```java
HDEL key field [field ...]
```

删除时会忽略不存在的域，返回被删除的域的数量，其中不包含被忽略的域

### HEXISTS:判断哈希表中的域是否存在

```java
HEXISTS key field
```

如果这个field存在，则返回1；如果这个哈希表key不存在，或field不存在，则返回0


