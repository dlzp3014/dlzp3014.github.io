---
layout: post
title: "KafkaProducer参数配置"
date: 2022-04-01 22:13:00
categories: Kafka
tags: Kafka
---

* content
{:toc}
Producer端大部分参数都有合理的默认值，一般不需要修改它们，只需要提供必要的`bootstrap.servers`、`key.serializer/value.serializer`参数，有些主要的参数如`ack`、`bacth.size`、`linger.ms`涉及Producer端的可用性和性能，在进行性能调优尤为重要

`在实际配置Producer参数过程中可以直接ProducerConfig类中定义好的名称`



## bootstrap.servers

用于指定生产者客户端连接Kafka集群所需的broker地址，格式为 hots:port

1. 可以设置一个或多个中间使用逗号隔开(生产者会从给定的broker里查找其他broker信息)
2. 建议至少设置两个以上broker地址，当其中任意一个宕机时，生产者仍然可以连接到Kafka集群

## key.serializer/value.serializer

用来指定key/value序列化操作的序列化器，必须填写全限定类名。

1. ProducerRecord<K, V>中泛型对应的就是消息中key和value的类型
2. 消息发往broker之前，需要将对应的key和value序列化为字节数组`【broker接收消息必须以字节数组的形式存在】`

## buffer.memory

用于缓存消息的缓冲区大小，默认32M。如果记录发送的速度超过了传送到服务器的速度，生产者将被阻塞，阻塞时长由`max.block.ms`参数决定，随后抛出异常。此参数大致对应于生产者占用的总内存(并不是生产者使用的所有内存都用于缓存：一些额外的内存用于压缩以及维护正在运行的`in-flight requests`)

备注：`Producer采用异步发送消息的架构`

1. Producer启动时会创建一块内存，用于缓冲待发送的消息
2. 消息的投递由专属线程负责从缓冲区读取消息执行发送，缓冲消息的内存空间即由buffer.memory参数指定
3. 缓冲区写入速度大于发送消息速度时，会造成缓冲区溢出，此时producer会阻塞消息的写入，等待超时后抛出异常并期望用户接入进行处理
4. 若Producer需要给多分区发送消息就需要考虑此参数以防止过小的内存缓冲区降低Producer的吞吐量

## compression.type

消息压缩类型，默认值为none，消息不会被压缩，可设置的参数为``gzip`, `snappy`, `lz4`(压缩/解压速度快), or `zstd(有着超高的压缩率，可达2.8)`

1. 压缩是整批的数据，所以批处理的效果也会影响压缩比(批处理越多，压缩效果越好)

2. 消息压缩可极大的减少网络带宽，降低网络IO传输，从而提高整体吞吐量，但会增加producer端cpu开销，如果broker端的压缩参数与Producer不同，broker端在写入消息时也会额外使用cpu资源对消息进行解压-重压操作

3. 消息压缩是一种使用时间换空间的优化方式，如果对延迟有一定要求，则不推荐对消息进行压缩

## retries/retry.backoff.ms

retries-用于配置消息瞬时发送失败(leader选举或网络抖动)后的重试次数，默认为0，不进行重试；retry.backoff.ms-用于设置两次重试之间的时间间隔，避免无效的频繁重试，默认100毫秒

1. 针对这种瞬时异常可以配置retries为大于0的值，以此通过Producer内部重试来恢复，重试达到设置次数会返回异常
2. 并不是所有的异常都可通过重试来解决，如消息太大，超过max.request.size参数配置
3. 配置retries/retry.backoff.ms之前，最好估算下可能的异常恢复时间，设定总的重试时间大于异常恢复时间，可以避免Producer过早的放弃重试
4. retires设置时需要注意以下两点：
   - 重试可会造成消息的重复发生：瞬时的网络抖动使得broker已成功写入消息但没有成功发生响应给producer，producer会认为消息发生失败，从而开启重试机制，为应对此风险Kafka要求consumer必须执行去重处理，Kafka 0.11.0.0版本后支持`exactly one`处理语义，从设计上避免类似的问题
   - 重试可造成消息的乱序：Kafka可以保证同一个分区中的消息是有序的，如果将acks参数设置为非0值，并且max.in.flight.request.per.connection参数设置为大于1的值`【Producer会将多个消息发送的请求缓存在内存中，默认为5个】`，重试就会出现乱序现象(第一批消息写入失败，第二批消息写入成功，重试第一批失败消息后成功，这两批次的消息就出现乱序)。一般而言，在需要保证消息顺序的场合下建议把max.in.flight.request.per.connection参数配置为1，Producer将确保某一时刻只能发生一个请求，此设置会影响整体的吞吐

## batch.size

用于设置batch中消息的大小，默认16KB，对Producer吞吐量和延迟有有着重要的作用

1. Producer会将发往同一个partition的多个消息封装为以batch中，当batch慢时，Producer会发生batch总的所有消息，此时就会产生一定的延迟
2. Producer并不总是等待batch满了才发生消息，可以通过liner.ms来设置消息发生延迟时间
3. 通常来说，一个较小的batch中包含的消息很少，一次发生请求能够写入的消息就会少，Producer的吞吐量就会低；由于不管能否填满batch，Producer都会为batch分配固定大小的内存，此时较大的batch会给内存使用带来极大的压力，因此batch.size参数的设置是一种时间与空间权衡的体现

## client.id

设定KafkaProducer对应的客户端id，不设置时KafkaProducer会自动生成一个非空字符串，内容为"producer-num"

## connection.max.idle.ms

用于设置多久之后关闭闲置的连接，默认9分钟

## delivery.timeout.ms

用于设置调用`send`方法后返回成功或失败的时间上限，默认2分钟

1. ​	此参数限制了消息在发送之前被延迟的总时间、等待broker确认的时间（如果可预期的话）以及允许可重试发送失败的时间
2. 如果遇到不可恢复的错误、重试次数已用尽或消息被添加到过期批次中，Producer可能会提前报告失败
3. 此配置的值应大于或等于request.timeout.ms、linger.ms时间的之和

## linger.ms

用于指定Producer发生ProducerBatch之前等待更多消息(ProducerRecord)加入ProducerBatch的时间，默认0，表示消息需要被立即发生，不需要等待batch被填满。

1. 控制消息发生延时行为
2. Producer在ProducerBatch被填满或等待时间超过linger.ms时发生出去
3. 增大此参数值会增加消息的延迟，但同时提升一定的吞吐量(Producer发送的每次请求中包含更多的消息，将请求的开销均摊到更多消息上)，是Producer吞吐量和延迟的权衡

## max.block.ms

用于配置 `KafkaProducer's send()`, `partitionsFor()`, `initTransactions()`, `sendOffsetsToTransaction()`, `commitTransaction()` and `abortTransaction()`方法的阻塞时长，默认1分钟

1. 对于`send()`方法，此超时限制了两次元数据获取和缓存分配的总等待时间(序列化、分区不计入此超时)
2. 对于`partitionsFor()`方法，此超时限制等待元数据的不可用时长
3. 事务相关的方法都是阻塞的，如果无法发现事务协调器transaction coordinator或在超时时间内没有响应，则可能超时

## max.request.size

用于限制Producer单次发送消息的最大值，默认1M。此参数还涉及一些其他联动的参数，如broker端的message.max.bytes，如果配置错误可能会一起一些不必要的异常（RecordTooLargeException），因此不建议盲目地增大这个参数的配置值

## partitioner.class

用于指定分区器，确定消息被发往哪个分区，可选参数

1. DefaultPartitioner，默认区分器，尝试`sticking to a partition`直到batch为空或者linger.ms等待结束。执行策略为如果没有指定分区但key存在则根据kay的哈希值选择一个分区；如果没有分区或key存在则当batch为空或者linger.ms等待接受选择的粘性分区将改变
2. RoundRobinPartitioner，将一系列连续消息中的每个消息发生到不同的分区中(与key的指定无关)，直到分区用完，然后重新开始。【Note：新建batch时会导致分配不均】
3. UniformStickyPartitioner，尝试`sticking to a partition`与key的指定无关，直到直到batch为空或者linger.ms等待结束
4. 实现Partitioner接口自定义分区器

## receive.buffer.bytes

读取数据时使用的TCP接收缓冲区(SO_RCVBUF)的大小。当取值为-1时，表示使用操作系统默认值

## request.timeout.ms

用于设置Producer等待请求响应的最长时间，默认30s，请求超时时可以选择重试。此参数需要比broker端`replica.lag.time.max.ms`值大，可减少重试导致消息重复的概率

## send.buffer.bytes

发送数据时使用的TCP发送缓冲区(SO_SNDBUF)的大小。当取值为-1时，表示使用操作系统默认值

## socket.connection.setup.timeout.max.ms

客户端等待socket连接建立的最长时间，默认30s。每次连接失败，连接设置超时将呈指数级增长，达到这个最大值。为了避免连接风暴，将对超时应用一个0.2的随机因子，从而产生一个比计算值低20%到高20%的随机范围

## socket.connection.setup.timeout.ms

客户端等待建立套接字连接的时间，默认10s，如果在超时之前没有建立连接，客户端将关闭套接字通道

## acks

用于指定分区中必须要有多少个副本收到这条消息之后生产者才会认为这条消息是成功写入的，即此参数控制Producer生产消息的持久性(durbility)，leader broker必须确保已成功写入消息的副本数， 是Producer中非常重要的参数，涉及消息的可靠性和吞吐量。默认值为1

对于Producer而言，Kafka在乎的是`已提交`消息的持久性，一旦消息被成功提交，那么只要有任何一个保存了该消息的副本存活，这条消息就会被视为`不会丢失`。Kafka所谓`已丢失`的消息其实并没有被成功写入Kafka，即消息没有被成功提交，Kafka对未成功提交消息的持久性不做任何保障，Producer可以通过回调机制处理发生失败的情况

消息成功提交：当Producer发生一条消息给Kafka时，这条消息会被发生到指定topic分区leader所在的broker，Procuder等待从该leader broker返回消息的写入结果(有超时时间)。

`Kafka能够保证consumer永远不会读取到尚未提交完成的消息`

leader broker何时发送写入结果给Producer是一个需要考虑的问题，这直接影响消息的持久性和Producer的吞吐量：Producer越快地接收到leader broker响应，就能越快地发送下一条消息，即吞吐量越大

acks有3个取值0、1、all，为`字符串类型`，配置如下：

```java
props.put(ProducerConfig.ACKS_CONFIG,"0");
```

- acks = 1：Producer发送消息后leader broker仅将消息写入本地日志，然后便发送响应结果给producer，无需等待ISR其他副本写入该消息。如果消息无法写入leader 副本，如在leader 副本副本崩溃、重新选举新的leader副本的过程中，生产者就会收到错误响应，为避免消息丢失，生产者可以选择重新发送消息。如果消息写入leader副本并返回成功响应给生产者，且被其他follower副本拉取之前leader副本崩溃，此时消息还是会丢失，新选举的leader副本并没有这条对应的消息。acks=1是消息可靠性和吞吐量之间的均衡
- acks = 0 ：Producer发送消息后不需要等待leader broker的响应，立即发送下一条消息。如果消息从发生到写入Kafka的过程中出现某些异常，导致Kafka并没有收到这条消息，Producer端无法通过回调机制感知发生过程中失败，消息也就会丢失，即`acks=0时producer不保证消息会被成功的发送`。在相同的配置环境中，acks=0时Producer的吞吐量时最大的
- acks = all 或-1 ：消息发生后，leader broker不仅会将消息写入本地日志，同时还需要等待ISR中所有其他副本都被成功写入到各自的本地日志后，才发生响应结果给Producer。在相同的配置环境中，acks=all可以达到最强的可靠性，但这并不意味着消息就一定可靠，因为ISR中可能只有leader副本，这就退化为acks=1的情况。要获得更高的消息可靠性还需要`min.insync.replicas`等参数的联动。具体参考：[Kafka消息的可靠性](/2022/04/14/kafka-message-reliable)


## enable.idempotence

是否开启幂等性，默认为true。具体参考：[Kafka消息的幂等性](/2022/04/14/kafka-idempotence)

## interceptor.classes

消息发往Kafka前，可对消息进行拦截处理。默认无拦截器

## max.in.flight.requests.per.connection

在阻塞之前，客户端在单个连接上发送的未确认请求的最大数量。如果此配置大于1，并且enable.idempotence=false，消息发生失败后重试时将导致消息乱序

## metadata.max.age.ms

在指定的周期内如果leader 副本没有发生变化，则强制刷新metadata数据，主动发现新的broker或者partitions。默认5m

## metadata.max.idle.ms

缓存空闲topic元数据的时间，Producer在超过缓存空闲topic元数据时，会将其遗忘，并且在下次访问给=该主题时强制执行该主题元数据的获取请求

## metric.reporters

## metrics.num.samples









