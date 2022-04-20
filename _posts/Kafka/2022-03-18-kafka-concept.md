---
layout: post
title: "Kafka相关概念"
date: 2022-03-18 21:15:00
categories: Kafka
tags: Kafka

---

* content
{:toc}


Kafka是一个多分区、多副本分布式消息系统，以高吞吐、可持久化、可水平扩容、支持流数据处理等特性被广泛使用

- 消息系统：系统解耦、冗余存储、流量削峰、缓冲、异步通信、可扩展、可恢复、`消息顺序性性保障及回溯消费`
- 存储系统：Kafak把消息持久化磁盘，有效降低数据丢失的风险(多副本)，可以把Kafka作为长期的数据存储系统来使用(通过数据保留策略或者启动主题日志压缩)
- 流式处理平台：为每个流式处理框架提供可靠的数据来源





## 简介

Kafka架构中包含多个Producer、多个Broker、多个Consumer，以及一个Zookeeper集群(2.8.0版本后废弃 )，其中Kafka用来复制集群元数据的管理，控制器的选举等操作。Producer将消息发生到Broker，Broker负责将收到的消息存储到磁盘中，Consumer负责从Broker订阅并消费消息。架构体系如下

![](/img/post.img/kafka/Kafka-system.png)



## 相关概念

- Producer:  生成者

发送消息一方，负责创建消息，将其投递到Kafka中

- Consumer：消费者

接收消息一方，连接到Kafka上并接收消息，从而进行相应的业务逻辑处理

- Broker：服务代理节点

Kafka服务节点或实例，多个Broker组成一个Kafka集群

- Topic：主题

Kafka中的消息以主题为单位进行归类，每一个消息都要指定一个主题。生产者负责将消息发送到特定的主题，消费者负责订阅主题并进行消费。`主题是一个逻辑上的概念`

- Partition：分区

主题可以被划分为多个分区，一个分区只属于一个主题(主题分区Topic-Partition)，同一主题下的不同分区包含不同的消息；，每个Partition都有自己的分区号(从0开始)，在存储层面上可以看作一个可缀加的日志(log)文件(有序不可修改的消息序列)，`分区是一个物理上的概念`，每条消息被发送到broker前，都会根据分区规则选择存储到哪个分区上，以解决单节点I/O压力，提升系统的吞吐量；主题创建时可通过指定的参数来设置分区的个数或者修改分区的数量，从而实现主题的水平扩容

- Replica：副本

Kafka为分区引入多副本机制，以提供容灾能力。同一分区的不同副本中保存相同的消息(同一时刻副本之间并非完全一样)，副本之间是`一主多从`关系，leader负责处理读写请求，follower只负责与leader的消息同步；Replica处于不同的Broker中，当leader出现故障时，Kafka会从follower中重写选举新的leader对外提供服务。Kafka多副本架构如下：

![](/img/post.img/kafka/Kafka-partition-replica.png)

- AR：Assigned Replicas

分区中的所有副本统称为AR

- ISR：In-Sync Replicas

所有与leader保持一定程度同步的副本，包括leader副本。ISR集合是AR集合中的一个子集，消息会先发送到leader，然后follower才能从leader中拉取消息进行同步，同步期间内follower相对于leader而言有一定的滞后(可通过参数进行配置)

- OSR：Out-of-sync Replicas

与leader同步滞后过多的副本，不包括leader 。 AR、ISR、OSR三者的关系为：AR=ISR+OSR。正常情况下，所有的follower都应该与leader保持一定程度的同步，即AR=ISR，OSR基本为空。

leader副本负责维护和跟踪ISR集合中所有follower副本的滞后状态，当follower副本落后太多或者失效时，leader副本会把它从ISR集合中移除到OSR集合中，如果OSR集合中的follower副本"追上"leader副本，leader副本会把它从OSR集合中转移到ISR集合中。默认情况下，当Leader副本发生故障时，只有在ISR集合中的副本才有资格被选举为新的leader，OSR集合中的副本没有任何机会(此原则可以通过修改参数配置来改变)。`Kafka承诺只要ISR集合中至少存在一个replica，那些已提交的消息就不会丢失`

- offset:偏移量

消息在被缀加到分区日志文件时会分配一个唯一标识`偏移量offset`；offset是从0开始顺序递增的整数，Kafka通过它来保证消息在分区内的顺序性，即Kafka保证消息在`分区有序而不是主题有序`，通过offset可以唯一定位到某个Partition下的一条消息；Kafa消费者端也有偏移量offset，此offset会随着消费进度不断前移，但不可能超过该分区最新消息的offset

【备注】：Kafka消费端也具备一定的容灾能力，Consumer使用Pull模式从服务器拉取消息并保存消费的具体位置，当消费者宕机后恢复上下时可以根据之前保存的消费位置offset重新来气需要的消息进行消费，这样就不会造成消息丢失

- HW：High Watermark 高水位

标识一个特定的消息偏移量offset，消费者只能拉取到这个offset之前的消息。ISR集合中最小的LEO为分区的HW，对消费者而言只能消费HW之前的消息

- LEO：Log End Offset

标志当前日志文件中下一条待写入消息的offset，LEO的大小相当于当前日志分区中最后一条消息的offset值+1。分区ISR集合中的每个副本都维护自身的LEO，ISR集合中最小的LEO即为分区的HW，对消费者而言只能消费HW之前的消息

offset、hw、leo关系如下：

![](/img/post.img/kafka/Kafka-offset-hw-leo.png)

## 示例说明ISR、HW、LEO之间关系

如 user-message 主题的某个分区的ISR集合有3个副本，即一个leader副本和2个follower副本，此时分区的LEO和HW都为3，消息3、从生产者发出之后会被先存入leader副本：

![](/img/post.img/kafka/Kafka-write-message-01.png)

在消息写入leader副本之后，follower副本会发送拉取请求，同步消息3,4

![](/img/post.img/kafka/Kafka-write-message-02.png)

在同步过程中，不同的follower副本的同步效率不同，如在某一时刻follower1完全跟上了leader副本而follower2只同步了消息3，此时leader副本的LEO为5，follower1的LEO为5，follower2的LEO为4，此时当前分区的HW取最小值4，消费者可消息到的offset为0到3之间的消息

![](/img/post.img/kafka/Kafka-write-message-03.png)

所有副本都成功写入消息3，4，整个分区的HW和LEO都为5，此时消费者可以消费到offset为4的消息

![](/img/post.img/kafka/Kafka-write-message-04.png)

Kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制。事实上，`同步复制要求所有能工作的follower副本都复制完，这条消息才会被确认为已成功提交`，这种复制方式极大地影响了性能；在异步复制方式下，follower副本异步地从leader副本中复制数据，数据只要被leader副本写入就被认为已经成功提交，在这种情况下，如果follower副本还没有将消息同步完，leader副本宕机，则会造成数据丢失。Kafka使用ISR的方式有效地权衡了数据可靠性和性能之间的关系



