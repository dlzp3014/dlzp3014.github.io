---
layout: post
title:  "Redis命令-键(key)命令"
date: 2019-10-07 08:38:00
categories: Redis 
tags: Redis-Command
---

* content
{:toc}

Redis的键命令主要用于管理Redis的键，如删除键，查询键，修改键及设置某个键等





## 查询键

### EXISTS：判断键是否存在

```java
EXISTS key
```

EXISTS命令用于判断指定的key是否存在，如果key存在则返回1，否则返回0


### KEYS:查找键

```java
KEYS pattern
```

KEYS命令用于按照指定的模式(pattern)查找所有的key。参数pattern类似于正则表达式：

- KEYS * :匹配查找所有key

- KEYS r?dis:`?`表示任意一个字符

- KEYS r\*dis:`*`表示0->n个字符

- KEYS r[ae]dis:·`[x...]` 表示匹配`[]`中的任意字符


【Note】:遇到特殊符号需要使用`\`隔开(转义)


### OBJECT:查看键的对象

```java
OBJECT subcommand [arguments [arguments]]
```

OBJECT命令用于从内部查看key的Redis对象，该命令通常用于在除错或者为了节省空间而对key使用`特殊编码`的情况下。如果要用Redis来实现与缓存相关的功能，则可以使用OBJECT命令来决定是否清除key

OBJECT命令有如下子命令：

- OBJECT REFCOUNT key 用于返回给定key`引用所存储的值的次数`，多用于除错

- OBJECT ENCODING key 用于返回给定key所存储的值所使用的`底层数据结构`

- OBJECT IDLETIME key 用于返回给定key自存储以来的`空闲时间`，以秒为单位


Redis对象具有多种编码格式：

- 针对字符串可以被编码为raw(一个字符串)或int(Redis会将字符串表示的64位有符号整数编码为整数来存储，依次来节约内存)

- 针对列表可以被编码为ziplist或linkedlist。ziplist是压缩列表，用来表示占用空闲较小的列表


- 针对集合可以被编码为intset或者hashtable。intset是只存储数字的小集合的特殊表示

- 针对哈希表可以被编码为zipmap或hashtable。zipmap是小哈希表的特殊表示

- 针对有序结合可以被编码为ziplist或skiplist。ziplist主要用于表示小的有序集合，而skiplist可以表示任意大小的有序集合


OBJECT命令的子命令REFCOUNT和IDLETIME会返回数字，而ENCODING会返回相对应的编码类型


### RANDOMKEY:随意返回一个键

```java

RANDOMKEY 
```

RANDOMKEY用于`随机返回当前数据库中的一个key`，并且不会删除这个key。如果数据库不为空，则将返回一个随机key，否则返回nil


## 修改键

### RENAME:修改键的名称

```java
RENAME key newkey
```

RENAME用于修改key的名称，将key的名称修改为newkey，如果这个key不存在，则返回一个错误，如果newkey已经存在，则RENAME命令执行后将会覆盖旧值


### RENAMENX:修改键的名称

```java
RENAMENX key newkey
```

RENAMENX命令用于修改key的名称，将key的名称修改为newkey，`当且仅当newkey不存在时才能修改`。如果key不存在，则返回一个错误。`RENAMENX命令成功执行后，返回1，表示key的名称修改成功，如果newkey已经存在，则返回0`


## 键的序列化

### DUMP:序列化键

```java
DUMP key
```

DUMP命令用于序列化给定的key，并返回被序列化的值。可以使用RESTORE命令来反序列化这个key

使用DUMP命令序列化生成的值具有以下特点：

- 这个值是具有64为的校验和，用于检测错误。RESTORE命令在进行反序列化之前，先会检查校验和

- 这个值的编码格式和RDB文件的编码格式保持一致

- RDB版本会被编码在序列化值中。如果Redis版本不同，那么这个RDB文件会存在不兼容，Redis也就无法对这个值进行反序列化

- 这个序列化的值中没有生存的时间信息


### RESTORE:对序列化值进行反序列化

```java
RESTORE key ttl serialied-value [REPLACE]
```

用于将一个给定的序列化值反序列化，并为它关联给定的key。参数ttl用于为key设置生存时间，单位为毫秒。如果参数ttl的值为0，则不设置生存时间

`RESTORE在执行反序列化操作之前，会先对序列化的RDB版本和数据校验和进行检查`。如果RDB版本不相同或者数据不完整，那么反序列化会失败，RESTORE拒绝进行反序列化

如果key已经存在，并且给定了REPLACE参数，那么使用反序列化得到的值来替换key的旧值；如果key已经存在，而没有设置REPLACE参数，则将返回一个错误。RESTORE命令执行成功时会返回ok


## 键的生存时间


### PTTL:获取键的生存时间(毫秒)

```java
PTTL key
```

用于以毫秒的形式返回key的剩余生存时间，与TTL命令相似。如果PTTL命令执行成功后，会返回key的剩余生存时间，单位毫秒；`如果key不存在，则返回-2；如果key存在，但是并没有设置生存时间，则返回-1`

### TTL：获取键的生存时间(秒)

```java
TTL key
```

用于返回key的剩余生存时间，以秒为单位。TTL命令执行后的返回结果与PTTL命令相同，只是单位为秒


### EXPIRE:设置键的生存时间(秒)

```java

EXPIRE key seconds
```

用于设置key的生命周期(生存时间)。`当key的生存时间为0(过期)时，这个key将会删除`：

- 如果想删除这个key的生存时间，则可以使用DEL命令连同这个key一起删除

- 如果想修改这个带有生存时间的key的值，则可以使用SET或GETSET命令来实现。`SET或者GETSET命令仅仅用于修改这个key的值，它的生存时间不会被修改 `

- 如果使用RENAME命令来修改一个key的名称，那么`改名后的key的生存时间和改名前的key的生存时间一样`

- 如果只是删除某个key的生存时间，而不想删除这个key，则可以使用`PERSIST命令来实现让这个key成为一个持久的key`


还可以使用EXPIRE命令来修改一个已经带有生存时间的key的生存时间，`旧的生存时间会被新的生存时间替换(更新生存时间)`


### PEXPIRE:设置键的生存时间(毫秒)

```java
PEXPIRE key milliseconds
```

PEXIPIRE命令用于以毫秒的形式设置key的生存时间，与EXPIRE命令相似


### EXPIREAT:设置键的生存UNIX时间戳(秒)

```java
EXPIREAT key timestamp
```

EXPIREAT命令用于为key设置生存时间，参数timestamp是UNIX时间戳，该命令的用法与EXPIRE命令的用法类似。如果生存时间设置成功，则返回1；当key不存在或者不能为key设置生存时间时，返回0


### PEXPIREAT:设置键的生存UNIX时间戳(毫秒)

```java
PEXPIREAT key milliseconds - timestamp
```

PEXPIREAT命令用于以毫秒为单位设置key的过期UNIX时间戳，与EXPIREAT命令相似


## 键值对操作


### MIGRATE:转移键值对到远程目标数据库

```java
MIGRATE host port key destination-db timeout [copy] [REPLACE]
```

MIGRATE命令用于将key`原子性地从当前数据库转移(复制)到指定的目标数据库中`，一旦转移成功，key就会出现在目标数据库中，当前数据库中的key就会被删除。MIGRATE命令是原子操作，它在执行的时间会阻塞进行转移的两个数据库，直到转移成功或者失败，又或者出现等待超时

MIGRATE命令的实现原理为：在当前数据库(源数据库)中，对给定的key执行DUMP命令，将它序列化后，转移到目标数据库中，目标数据库再使用RESTORE命令对数据进行反序列化，将反序列化后的结果保存到数据库中，源数据库就好像目标数据库的客户端一样，只要遇到RESTORE命令返回OK，就会调用DEL命令删除自己数据库中的key


参数timeout：用于设置当前数据库与目标数据库进行转移时的最大时间间隔，单位为毫秒。当转移的时间超过了timeout时，就会报请求超时


MIGRATE命令需要在timeout时间范围内完成I/O操作。如果在转移数据的过程中发生I/O操作或者达到了超时时间，那么该命令将会终止执行，并返回一个特殊的错误：IOERR。出现IOERROR有两种情况：

- 源数据库和目标数据库中可能同时存在这个key

- key可能只在源数据库存在


`MIGRATE命令执行时key是不可能发生丢失的：在执行的过程中出现了错误，该命令可以保证这个key只存在于源数据库中`

参数copy:如果在MIGRATE命令中设置了COPY参数，则表示在`转移之后不会删除源数据库中的key`

参数REPLACE:如果在MIGRATE命令中设置了REPLACE参数，则表示在转移过程中会替换目标数据库中已经存在的key


操作实例：

```java
# 启动两个数据库实例
./redis-service --port 6379 &
./redis-service --port 6380 &

# 客户端 6379

./redis-cli -p 6379

SET article 'message ...'

MIGRATE 127.0.0.1 6380 article 0 10  //0 为数据库编号

# 客户端 6380

./redis-cli -p 6380

GET article
```

### MOVE:转移键值对到本地目标数据库

```java
MOVE key db
```

MOVE命令用于将当前数据库中的key转移到指定的数据库db中。如果当前数据库中没有指定的key，那么MOVE命令什么也不做；如果当前数据库和给定数据db中存在相同的key，那么MOVE命令没有任何效果。可以将MOVE命令的这一特性看作锁源语

```shell

MOVE article 3 # 将article键值对转移到3号数据库中

GET article 
(nil)

SELECT 3 # 切换到3号数据库

get article 

```

### SORT：对键值对进行排序

```java

SORT key [BY pattern] [LIMIT offset count] [GET pattern [get pattern ... ]] [ASC|DESC] [ALPHA] [STORE destination]

```

SORT命令主要用于排序，它返回或保存给定列表、集合、有序集合key中经过排序的元素。排序默认以数字作为对象，值会被解释为double类型的浮点数，然后进行比较。如果没有使用STORE参数化，那么排序结果将以列表的形式返回；如果使用了STORE参数，则将返回排序结果的元素数量。具体使用过程请查看:[Redis命令-排序(STOT)命令](/2019/10/10/redis-sort-cmd/)


### TYPE命令

```java
TYPE key
```

TYPE用于返回key所对应值的类型，返回值如下：

- key不存在，返回none
- key所对应的值时字符串类型，返回string
- key所对应的值是列表类型，返回list
- key所对应的值是集合类型，返回set
- key所对应的值是有序集合类型，返回zset
- key锁对应的值是哈希类型，返回hash


## 删除键

### DEL:删除键

```java
DEL key [key ...]
```

用于删除给定的一个或者多个key，如果这个key不存在，则会忽略删除，返回被删除key的数量

### PERSIST:删除键的生存时间

```java
PERSIST key 
```

PERSIST命令用于删除给定key的生存时间，将这个带有生存时间的key转化为一个不带生存时间的永久key(永不过期)。当PERSIST命令成功删除给定key的生存时间时返回1，如果key不存在或者key并没有生存时间，则返回0，表示该命令执行失败

