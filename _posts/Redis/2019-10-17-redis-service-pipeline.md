---
layout: post
title:  "Redis流水线(Pileline)"
date:   2019-10-17 08:38:00
categories: Redis 
tags: Redis-Serve
---

* content
{:toc}

Redis采用TCP协议来对外提供服务，基于Request/Respone请求响应的模式：客户端通过Socket连接发起请求，发生一条命令给服务器，等待服务器应答，进行处理后，返回结果。在这个过程中，每个请求在命令发出后会阻塞等待Redis服务器进行处理，处理完毕后才会将结果返回给客户端。

每条命令在发送与结果的遇中都会占用两个网络传输，在业务量庞大的情况下，如处理一个业务可能会对Redis做大量`连续的多个操作`，这些操作通常`依次联系`，如果每个连接、执行命令需要0.1s，那么在1s内也只能处理10个命令，在实际应用中是不能满足需求的，这将严重影响Redis的性能(网络传输耗时也是一个严重的问题)。为解决此类问题，Redis引入了流水线(Pipeline)，也可以成为管道



## Pipeline技术

Pipeline技术就是`一次性把所有的命令请求发生给服务器`，这样就可以避免频繁地发生、接收命令所带来的网络延迟，减少I/O调用次数。`服务器在接收到一堆命令后，会依次执行，然后把结果打包，一次性返回给客户端`。使用Pipeline技术的好处就是节约网络宽带，缩短访问时间，减少服务器I/O调用次数，提供Redis的性能


【Note】: `需要注意的是要控制Pipeline的大小，也就是它每次最多可以发送多少条命令的限制。过多使用将会消耗Redis的内存，并且Pipeline一次只能运行在一个Redis节点上`



## Client使用Pipeline


- 不使用Pipeline执行1000次HSET操作：

```java
Jedis jedis=new Jedis("127.0.0.1",6379);

//循环1000次，每次执行hset命令插入
for (int i=0;i<1000 ;i++ ) {
	jedis.hset("pipeline:test",i);
}

```

- 使用Pipeline技术执行100此HSET操作

```java
Jedis jedis=new Jedis("127.0.0.1",6379);

//选项10次，使用pipeline每次执行100条命令
for (int i=0;i<10 ;i++ ) {
	
	Pipeline pipeline=jedis.pipeline(); //激活pipeline

	for(int j=i*100;j<(i+1)*100;j++){ //每次同时发生100条命令给服务器
		pipeline.hset("pipeline:test",i);
	}
	pipeline.syncAndReturnAll(); //同步执行pipeline,返回所有结果集，
}
```

分别执行测试代码，可以看出，`使用Pipeline技术，极大地缩短了命令的执行时间，提升了Redis的性能`



