---
layout: post
title:  "Redis慢查询"
date:   2019-10-14 08:38:00
categories: Redis 
tags: Redis-Serve
---

* content
{:toc}


Redis慢查询日志功能用于记录服务器在执行命令时，超过给定时长的命令请求的相关信息，这些相关信息包括慢查询ID、发生时间戳、耗时、命令的详细信息。在实际应用中，开发人员和运维人员可以`通过慢查询日志来定位系统的慢操作`，然后利用这个功能产生的日志来`监视和优化查询速度`





## 慢查询配置

慢查询有两个重要的配置参数：

- 慢查询的预设阈值slowlog-log-slower-than

slowlog-log-slower-than参数是慢查询的预设阈值，`用于指定执行时间超过多少微妙的命令请求会被记录到日志中(1s = 1000 000μs)`。例如，如果这个参数的值为1000时，那么执行时间超过1000μs的命令就会被记录到慢查询日志中

- 慢查询日志的长度slowlog-max-len参数表示慢查询

slowlog-max-len参数表示慢查询的长度，用于指定服务器最多保存多少条慢查询日志，Redis服务器使用先进先出的方式保存多条慢查询日志。当服务器存储的慢查询日志数量等于slowlog-max-len参数的值时，服务器在添加一条新的慢查询日志之前，会先将最旧的的一条慢查询日志删除。例如，slowlog-max-len参数的值为1000，假设服务器此时已经存储了1000条慢查询日志，如果服务器还将继续存储一条慢查询日志，那么它会先删除目前保存的最旧的那条日志，再把新日志添加进去


在Redis的配置文件redis.conf中，慢查询参数的默认配置如下：

```java
slowlog-max-slower-than 10000 #默认为10s
slowlog-max-len 128 #默认为128
```

可以通过`CONFIG SET`命令来修改这两个参数。如使用CONFIG SET 命令将slowlog-max-slower-than 参数的值设为0μs，表示Redis服务器执行的任何命令都会被记录到慢查询日志中

```java
CONFIG SET slowlog-max-slower-than 0
```

可以使用`CONFIG GET slowlog*`命令来查看慢查询的配置：

```java
CONFIG GET slowlog*
```

在慢生成慢查询日志后，可以使用SHOWLOG GET 命令来查看操作命令的请求:

```java
SHOWLOG GET
```

## 慢查询生命周期

Redis执行一条命令需要经过发生命令、命令排队、命令执行，返回结果等几个步骤，而慢查询日志功能则发生在命令执行的过程中。客户端发生命令给Redis服务器，Redis服务器需要对慢查询进行排队处理。在命令执行的过程中，如果命令执行超时，就会被写入慢查询日志中，执行完后返回结果，这就是慢查询的声明周期


## 慢查询日志

- 获取慢查询日志：

使用SLOWLOG GET 命令获取慢查询日志，可以为该命令设置一个参数n，指定获取多少条慢查询日志：

```java

127.0.0.1:6379> set countname dlzp
OK
127.0.0.1:6379> slowlog get
1) 1) (integer) 1  # 慢查询日志的唯一标识符(UID)，也就是慢查询ID
   2) (integer) 1571154752 # 命令执行后的UNIX时间戳
   3) (integer) 4234 # 命令执行的时长，以微妙计算
   4) 1) "set" # 执行的命令及命令参数
      2) "countname"
      3) "dlzp"
```

每条慢查询日志中由4个属性组成，分别是慢查询ID、UNIX时间戳、命令执行时长、命令及命令参数。每条慢查询日志的ID都是唯一的，而且不会被重复设置，只会在Redis重启之后重新设置它

`慢查询日志也存储在内存中，没有专门的日志文件来存储慢查询日志内容，所以在获慢查询日志的时候，速度会比较快`。伪代码如下：

```c
def SHOWLOG_GET(number=None):
	if number is None:
		number = SHOWLOG_LEN() # 链表长度，全部慢查询日志

	for log in redisServer.slowlog:
		if number <=0:
			break
		else:
			number -=1 #计数器值减一
		printf(log)

def SHOWLOG_LEN():
	return len(redisServer.slowlog) # slowlog链表的长度
```

另外，用于清除所有慢查询日志的SLOWLOG RESET命令可以用如下伪代码类定义：

```c
def SHOWLOG_RESET():
	for log in redisServer.slowlog:
		deleteLog(log)
```

- 保存慢查询日志：


服务器状态中包含了几个和慢查询日志功能相关的属性：


```c
struct redisServer{

	long long slowlog_entry_id; //下一条慢查询日志的ID，初始值为0

	list *slowlog;  //保存了所有慢查询日志的链表

	long long slowlog_max_slower_than; //服务器配置 slowlog-max-slower-than选项的值

	unsigned slowlog_max_len; //服务器配置slowlog-max-len选项的值

};

```

slowlong链表保存了服务器中的所有慢查询日志，链表中的每个节点都保存了一个slowlogEntry结果，每slowlogEntry结果代表一条慢查询日志：

```c
typedef struct  slowlogEntry
{
	long long id; //慢查询日志的ID，唯一标识符

	tiem_t time; //命令执行的时间，格式为UNIX时间

	long long duration; //执行命令消耗的时间，以微妙为单位

	robj **argv; //命令与命令参数

	int argc; //命令与命令参数的数量
} slowlogEntry;
```

在每次执行命令的之前和之后，程序后悔极了微妙格式的当前UNIX时间戳，这两个时间戳之间的差就是服务器执行命令所耗费的时长，服务器会将这个时长作为参数之一传给是否需要为这次执行的命令创建慢查询(slowlogpushEntryIfNeeded函数)，伪代码如下：

```c
before = unixtime_now_in_us() # 记录执行命令前的时间

execute_command(argv,argc,client) # 执行命令

after = unixtime_now_in_us(); # 记录执行命令后的时间

slowlogpushEntryIfNeeded(argv,argc,before-after) # 检查是否需要创建新的慢查询日志

```

slowlogpushEntryIfNeeded函数有两个作用：

1)检查命令的执行时长是否超过slowlog-max-slower-than选项所设置的时间，超过时就为命令创建一个新的日志，并将新日志添加到slowlog链表的表头

2)检查慢查询日志的长度是否过程slowlog_max_len选项所设置的长度，超过时将多出现的日志从slowlog链表中删除

slowlogpushEntryIfNeeded伪代码：

```c
void slowlogpushEntryIfNeeded(robj **argv,int argc,long long duration){

	if(sever.slowlog_max_slower_than < 0) return; //慢查询功能未开启，直接返回

	if(duration >= slowlog_max_slower_than) //执行时间超过服务器设置上限，将命令添加到慢查询日志
		listAddNodeHead(server.slowlog,slowlogCreateEntry(argv,argc,duration));

	while(listLength(server.slowlog)>server.slowlog-max-len) //如果日志数量过多，删除
		listDelNode(server.slowlog,listLast(server.slowlog));
}

```

## 慢查询日志总结

- 慢查询日志命令：

SLOWLOG GET N: 获取N条慢查询日志

SLOWLOG LEN: 获取慢查询日志的长度

SLOWLOG RESET: 清空慢查询日志

- 慢查询日志实现：

Redis的慢查询日志功能用于记录执行时间超过指定时长的命令，服务器会将所有的慢查询日志保存在服务器状态的slowlog链表中，每个链接节点都包含了一个slowlogEntry结果，每个slowlogEntry结果代表一条慢查询日志；打印和删除慢查询日志可以通过遍历slowlog链表来完成，链表的长度就是服务器保存慢查询日志的数量；新的慢查询日志会被添加到slowlog链表的表头，如果日志的数量超过slowlog-max-len选项的值，多出来的日志会被删除

- 应用中对于慢查询日志应该注意：

1) `slowlog-max-len 参数的值不要设置得太小`。在记录慢查询日志时，`Redis会对长命令进行截断处理`，并不会占用大量的内存，默认128.建议设置为1000，原因在于慢查询日志存储在内存中，`如果该参数设置得过小，会导致之前的慢查询日志丢失`


2) slowlog-max-slower-than 参数的值不要设置得太大，默认设置为10000微妙，也就是10ms，表示命令执行时长超过10ms就会被判定为慢查询。`在实际应用中也只有1ms或2ms，需要根据Redis的并发量来设置`。`Redis采用单线程响应命令，对于高并发、高流量的场景，如果命令执行的时长超过1ms，那么Redis最多可以支撑的OPS(每秒查询率)不到1000条，所以对于高并发、高流量的场景，建议将该参数设置为1ms`


3) `定期对慢查询进行持久化，也就是定期保存备份`





