---
layout: post
title: "Kafka日志存储"
date: 2022-03-27 01:14:00
categories: Kafka
tags: Kafka
---

* content
{:toc}
Topic是逻辑上的概念，而partition时物理上的概念，每个partition对应于一个log文件，该文件存储了Producer生产的数据。Producer产生的数据会被不断追加到该log文件末，为防止log文件过大，Kafka采取了分片和索引机制，将每个partition分为多个segment，每个segment包括： .index(偏移量索引文件) 、.log(日志文件) 、.timeindex(时间戳所有文件)等文件，这些文件存于一个文件夹下，文件夹命名规则为：topic+分区号。其中index和log文件以当前segment的第一条消息的offset命名的




## 日志清理







