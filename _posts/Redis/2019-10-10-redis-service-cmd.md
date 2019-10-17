---
layout: post
title:  "Redis命令-服务器命令"
date:   2019-10-10 08:38:00
categories: Redis 
tags: Redis-Command
---

* content
{:toc}

Redis服务器命令主要用于操作管理Redis服务，如管理Redis的日志、保存数据、修改相关配置。具体使用如下：




## 管理客户端

### CLIENT LIST:获取客户端相关信息

```java
CLIENT LIST
```

用于获取所有连接到服务器的客户端信息和统计数据。返回的域含义如下：

- id: 客户端编号
- addr: 客户端的地址(IP+PORT)
- fd: 套接字所使用的文件描述符
- name: 客户端连接的名字
- age: 已连接时长，单位为秒
- idle: 连接空闲时长，单位为秒
- flags: 客户端flag
- db: 客户端正在使用的数据库，数值表示数据库的索引号
- sub: 已订阅的消息频道数量 
- psub: 已订阅模式的数量
- multi: 在事务中被执行的命令数量
- qbuf: 查询缓冲区的长度，0表示没有分配查询缓冲区，单位为字节
- qbuf-free: 查询缓冲区剩余空间的长度，0表示没有剩余空间，单位为字节
- obl: 输出缓冲区的长度，0表示没有分配输出缓冲区，单位为字节
- oll: 输出列表中包含的对象数量。如果输出缓冲区没有剩余空间，则命令回复以字符串对象的形式被添加到这个队列中
- omem: 输出缓冲区和输出列表占用的内存总量
- events: 文件描述符事件。r在loop事件中，表示套接字是可读的;w在loop事件中，表示套接字是可写的
- cmd:最近一次执行的命令

客户端flag可以由以下几个部分组成：

- O: 客户端MONITOR模式下的附属节点(slave)
- S: 客户端是一般模式下(normal)的附属节点 
- M: 客户端是主节点(master)
- x: 客户端正在执行事务
- b: 客户端客户端正在等待阻塞事件
- i: 客户端正在等待VM I/O操作(已废弃)
- d: 一个受监控(watched)的键已被修改，EXEC命令将执行失败
- c: 在将恢复完整地写出来之后，关闭连接
- u: 客户端未被阻塞(unblocked)
- A: 尽可能地关闭连接
- N: 为设置任何flag

```java
CLIENT LIST
id=2 addr=127.0.0.1:58237 fd=9 name= age=21 idle=21 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=18446744073709537584 events=r cmd=command
```

### CLIENT GETNAME:获取客户端名字

```java
CLIENT GETNAME
```

用于获取连接设置的名字。如果是新创建的连接，则没有名字(nil)

### CLIENT SETNAME:设置客户端名字

```java
CLIENT SETNAME connection-name
```

用于为当前连接设置一个名字，执行CLIENT LIST命令后的结果中会有这个连接的名字，可以用来`区分`当前正在与服务器连接的客户端。使用字符串类型来保存这个连接的名字，最多可以占用512M的空间。`在为连接设置名字的时候，名字中要尽量避免出现空格`。可以通过将一个连接的名字设置为空字符串来实现`删除`这个连接的名字，在默认情况下，新创建的连接没有名字



### CLIENT PAUSE:在指定时间范围内停止来自客户端的命令

```java
CLIENT PAUE timeout

```

用于在指定时间范围内停止运行来自客户端的命令，参数timeout是一个正整数，以毫秒为单位。`当前命令会阻塞所有与客户端的交互，直到超时后才执行命令`


### CLIENT KILL:关闭客户端连接

```java
CLIENT KILL ip:port
```

用于关闭一个客户端连接，ip:port为这个客户端的IP地址和端口。Redis是单先吃的，当一个Redis命令在执行时，不会有客户端被断开连接



## 查看Redis服务器信息

### COMMAND：查看Redis命令的详细信息

```java
COMMAND
```

COMMAND命令以数组的形式返回Redis的所有命令的详细信息

### COMMADN COUNT:统计Redis的命令个数

```java
COMMADN COUNT

```
返回一个数值，表示Redis的命令个数


### COMMAND GETKEYS:获取指定的所有键

```java
COMMAND GETKEYS 
```


### COMMAND INFO：查看Redis命令的描述信息

```java
COMMAND INFO key [key ...]
```

### DBSIZE:统计当前数据库中键的数量

```java
DBSIZE
```
用于统计当前数据库中键的数量


### INFO:查看服务器的各种信息

```java

INFO [section]
```

INFO用于查看Redis服务器的各种信息及统计相关数据值。参数section的设置可以让INFO命令只返回某一部分的信息。INFO命令执行后，会返回如下几部分的信息：

- server: 主要说明Redis的服务器信息
- clients: 记录已连接客户端的信息
- memory: 记录了Redis服务器的内存相关信息
- persistence: 记录了与持久化(RDB持久化和AOF持久化)相关的信息
- stats: 记录了相关的统计信息
- replication: 记录了Redis数据库主从复制信息
- cpu： 记录了CPU的计算量统计信息
- commandstats: 记录Redis各种命令的执行统计信息，如执行命令消耗的CPU时间、执行次数等
- cluster: 记录了与Redis集群相关的信息
- keyspace: 记录了与Redis数据库相关的统计信息，如键的数量

参数section除了取上面的值以外，还可以是all(表示返回所有信息)和default(表示返回默认选项的信息)

```java
INFO ALL
#Server
redis_version:3.2.100
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:dd26f1f93c5130ee
redis_mode:standalone
os:Windows
arch_bits:64
multiplexing_api:WinSock_IOCP
process_id:102384
run_id:9d1c9e03999d83622df44278a6df9314c0450cc5
tcp_port:6379
uptime_in_seconds:4025
uptime_in_days:0
hz:10
lru_clock:10527478
executable:E:\Redis-x64-3.2.100\redis-server.exe
config_file:

#Clients
connected_clients:3
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0

# Memory
used_memory:730968
used_memory_human:713.84K
used_memory_rss:693152
used_memory_rss_human:676.91K
used_memory_peak:808136
used_memory_peak_human:789.20K
total_system_memory:0
total_system_memory_human:0B
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
mem_fragmentation_ratio:0.95
mem_allocator:jemalloc-3.6.0

# Persistence
loading:0
rdb_changes_since_last_save:0
rdb_bgsave_in_progress:0
rdb_last_save_time:1570804541
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok

# Stats
total_connections_received:8
total_commands_processed:46
instantaneous_ops_per_sec:0
total_net_input_bytes:1886
total_net_output_bytes:29313356
instantaneous_input_kbps:0.00
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:2
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0

# Replication
role:master
connected_slaves:0
master_repl_offset:0
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:0.20
used_cpu_user:0.08
used_cpu_sys_children:0.00
used_cpu_user_children:0.00

# Commandstats
cmdstat_llen:calls=2,usec=9,usec_per_call=4.50
cmdstat_keys:calls=3,usec=77,usec_per_call=25.67
cmdstat_dbsize:calls=1,usec=1,usec_per_call=1.00
cmdstat_info:calls=2,usec=167,usec_per_call=83.50
cmdstat_client:calls=25,usec=319,usec_per_call=12.76
cmdstat_command:calls=13,usec=2787,usec_per_call=214.38

# Cluster
cluster_enabled:0

# Keyspace
db0:keys=1,expires=0,avg_ttl=0
127.0.0.1:6379>

```

### LASTSAVE:获取最近一次保存数据的时间

```java
LASTSAVE
```
用于获取最近一次Redis成功将数据保存到磁盘上的时间，时间格式是UNIX时间戳

### MONITOR:实时打印服务器接收到的命令

```java
MONITOR
```
MONITOR命令常用于调试，它会实时打印出Redis服务器接收到的命令

### TIME:获取当前服务器的时间

```java

TIME
```
TIME命令执行后，会返回包含两个字符串的列表，第一个字符串是当前时间戳，第二个字符串是在当前这一秒内逝去的毫秒数



## 修改并查看相关配置

### CONFIG SET:修改Redis服务器的配置

```java
CONFIG SET parameter value
```

CONFIG SET 用于修改Redis服务器的配置，`修改之后不需要重新启动服务器也能生效，也可以使用该命令来修改Redis的持久化方式`。能够被CONFIG SET命令所修改的Redis配置参数都可以在配置文件redis.conf中找到

```shell
CONFIG SET showlog-max-len 1000 # 修改服务器慢日志的最大长度

```

### CONFIG GET:查看Redis服务器的配置

```java
CONFIG GET parameter
```

CONFIG GET命令用于获取运行中的Redis服务器的配置参数。`参数parameter是命令搜索关键字`，用于查找所有匹配的配置参数，查找返回的参数和值以键值对的形式排列：

- CONFIG GET * 列出Redis服务器的所有配置参数及参数值

- CONFIG GET a* 列表出Redis服务器所有以a开头的配置参数及参数值

```java
CONFIG GET au* #查看AOF的相关配置
```

### CONFIG RESETSTAT:重置INFO命令中的统计数据

```java
CONFIG RESETSTAT
```

用于重置INFO命令中的一些统计数据：

- Keyspace hit: 表示键空间命中次数

- Keyspace misses: 表示键空间不命中次数

- Number of commands processed: 表示执行命令的次数

- Number of connections received: 表示连接服务器的次数

- Number of expired keys: 表示过期key的数量

- Number of rejected connections: 表示被拒绝的连接数量

- Latest fork(2) time: 表示最后执行fork(2)的时间

- The aof_delayed_fsync count: 表示aof_delayed_fsync计数器的值


### CONFIG REWRITE:改写Redis配置文件

```java
CONFIG REWRITE
```

在Redis服务器启动时，会用到redis.config文件，可以使用CONFIG REWRITE命令来修改这个文件。CONFIG REWRITE命令的作用就是通过尽可能少的修改，尽量保留最初的配置，将服务器当前正在使用的配置信息保存到redis.conf文件中。如果启动Redis服务器时所指定的redis.conf文件不存在，则可以使用CONFIG REWRITE命令重新构建并生成一个新的redis.conf文件

`对redis.conf文件的修改是原子性的，并且具有一致性，要么成功，要么失败`。



## 数据持久化

### BGREWRITEAOF:执行AOF文件重写操作

```java
BGREWRITEAOF
```
用于执行一个AOF文件重写操作。该命令执行后，会创建一个当前AOF文件的优化版本。当BGREWRITEAOF命令执行失败时，AOF文件数据并不会丢失。旧的AOF文件在BGREWRITEAOF命令执行成功之前是不会被修改的，因此不存在数据丢失问题

`在Redis的高版本中，AOF文件的重写将由Redis自动触发，使用BGREWRITEAOF命令只是用于手动触发重写操作`


### SAVE:将数据同步保存到磁盘中

```java
SAVE
```

SAVE命令具体执行的是一个`同步保存操作`，它以RDB文件的形式将当前Redis的所有数据快照保存到磁盘中。在生成环境中，不建议使用SAVE命令来保持数据，因为其在执行后会`阻塞所有客户端`。`推荐使用BGSAVE命令来异步执行保存数据的任务`。如果后台子进程保存数据失败，或者出现其他问题，则可以使用SAVE命令来做最后的保存



### BGSAVE：将数据异步保存到磁盘中

```java
BGSAVE
```

在Redis服务后端采用异步的方式将数据保存到当前数据库的磁盘中。BGSAVE命令的执行原理为：在BGSAVE命令执行后会返回OK，之后Redis启动一个新的子进程，原来的Redis进程(父进程)继续执行客户端请求操作，而子进程则负责将数据保存到磁盘中，然后退出。可以使用LASTSAVE命令来查看相关信息，进而判断BGSAVE命令是否将数据保存成功


## 主从服务

### SYNC

Redis复制功能的内部命令

```java
SYNC
```

### PSYNC

```java
PSYN <MASTER_RUN_ID> <OFFSET>
```

### SLAVEOF

在Redis运行时，可以使用SLAVEOF命令动态修改复制功能的行为。SLAVEOF host port命令来修改当前服务器使其转变为指定服务器的从属服务器(Slave Server)，如果当前服务器是某个主服务器的从属服务器，则在执行其命令后，会使当前服务器停止对旧主服务器的同步，并且将旧数据集丢弃，然后开始对新主服务器数据进行同步

如果不想在同步时丢失数据集，则可以使用SLAVEOF NO ONE命令，该命令执行后不会丢弃同步数据集。当主服务器出现故障的时候，可以利用该命令将从属服务器用作新的主服务器，实现数据不丢失，不间断运行


### ROLE

查看主从服务器的角色：master、slave、sentinel
```java
ROLE
```

## 服务器管理

### SHOWLOG:管理Redis的慢日志

```java
SHOWLOG subcommand [argument]
```

`Slow log(慢日志)是Redis的日志系统，用于记录查询执行时间。查询执行时间指的是执行一个查询命令所耗费的时间，它不包括客户端响应、发生信息等I/O操作、Slow log保存在内存里，读写速度非常快`

```java
SHOWLOG GET 3 #查看日志信息

SHOWLOG LEN # 查看当前慢日志的数量

SHOWLOG RESET # 清空慢日志
```

### SHUTDOWN:关闭Redis服务器或者客户端

```java
SHUTDOWN [SAVE|NOSAVE]
```

SHUTDOWN命令具有多种作用：

- 直接关闭Redis服务器

- 关闭(停止)所有客户端

- 在AOF选项被打开的情况下，执行SHUTDOWN命令将会更新AOF文件

- 如果Redis服务中至少存在一个保存点在等待，则在执行SHUTDOWN命令的同时将会执行SAVE命令

在持久化被打开的情况下，执行SHUTDOWN命令，它会保存服务器正常关闭，不会丢失任何数据

SHUTDOWN NOSAVE作用域SHUTDOWN SAVE命令的作用刚好相反，它会阻止Redis数据库执行保存操作，即使设置了一个或多个保存点也会阻止。`SHUTDOWN命令执行成功后任何信息也不返回，此时服务器和客户端的连接会断开，同时客户端自动退出`
