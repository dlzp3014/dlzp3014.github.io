---
layout: post
title:  "Redis命令-连接命令"
date: 2019-10-10 08:38:00
categories: Redis 
tags: Redis-Command
---

* content
{:toc}


Redis连接命令主要用于连接Redis的服务，如查看服务状态、切换数据库等。具体操作如下：





## AUTH:解锁密码

```java
AUTH password
```

AUTH命令用于解锁密码，可以通过修改Redis配置文件中requirepass项的值来为Redis设置密码，进而使用密码来保护Redis服务器，命令为：

```java
CONFIG SET requirepass password
```

在密码设置成功之后，每次连接Redis服务器都需要使用AUTH命令来解锁密码，解锁成功之后才能使用Redis的其他命令。

```java
CONFIG SET requirepass 123456 # 设置数据库密码

./redis-cli

AUTH 123456 #认证

CONFIG GET requirepass # 查看数据库密码
```

可以使用CONFIG SET requirepass命令来将密码设置为空，也就是清空密码


## QUIT：断开连接

```java
QUIT
```
用于断开当前客户端与服务器的连接。一旦所有等待中的回复顺序写入客户端，这个连接就会被断开


## PING:查看服务器的运行状态

```java
PING
```

用于查看Redis服务器是否正常运行，如果服务器正常运行则会返回PONG

## ECHO:打印消息

```java
ECHO message
```

用于输出打印消息，主要用于测试


## SELECT:切换数据库

```java
SELECT index
```

SELECT命令用于切换数据库，其中index是数字值，为数据库的索引，从0开始，默认使用0号数据库。`Redis数据库默认从0号到15号，超过将将会报错`




