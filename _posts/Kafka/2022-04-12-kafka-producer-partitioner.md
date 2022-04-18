---
layout: post
title: "KafkaProducer分区器"
date: 2022-04-12 22:13:00
categories: Kafka
tags: Kafka
---

* content
{:toc}
Kafka Producer发送消息过程中一个重要的步骤就是要确定将消息发生到指定topic的哪个分区中，构建ProducerRecord对象时，如果指定partition属性值，则不需要分区器作用，partition代表的就是所要发往的分区号；如果没有指定，则需要依赖分区器，将消息投递到对应的分区中。

`分区器的作用就是为消息分配分区`






## Partitioner接口定义

Partition接口继承了了Configurable接口，此接口中只包含一个方法`configure(Map<String, ?> configs)`，主要用来获取配置信息及初始化数据，Partitioner接口定义源码如下：

```java
public interface Partitioner extends Configurable, Closeable {

    /** 用来计算分区号
     * Compute the partition for the given record. 
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes The serialized key to partition on( or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes The serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);

    /** 分区关闭时释放资源
     * This is called when partitioner is closed.
     */
    void close();

    /** 通知分区将创建新的batch
     * Notifies the partitioner a new batch is about to be created. When using the sticky partitioner,
     * this method can change the chosen sticky partition for the new batch. 
     * @param topic The topic name
     * @param cluster The current cluster metadata
     * @param prevPartition The partition previously selected for the record that triggered a new batch
     */
    default void onNewBatch(String topic, Cluster cluster, int prevPartition) {
    }
}
```



## Kafka中提供的实现

### DefaultPartitioner

Kafka中提供的默认分区器为DefaultPartitioner：如果Key不为null，默认的分区器会对key进行hash(采用NurmurHash2算法，具备高运算性能及低碰撞率)，最终根据得到的hash值来计算分区号，拥有相同key的消息会被写入同一个分区；如果key为null时。消息将会以轮训的方式发往主题内的各个可用分区。

- key不为null，计算得到的分区号会是`所有分区中的任意一个`
- key为null且有可用分区时，计算得到的分区号为`可用分区中的任意一个`

在不改变主题分区数量的情况下，key与分区之间的映射可以保持不变，如果主题中增加了分区，就难以保证key与分区之间的映射关系

DefaultPartitioner源码如下：

```java
/**
 * The default partitioning strategy:
 * <ul>
 * <li>If a partition is specified in the record, use it 指定分区直接使用
 * <li>If no partition is specified but a key is present choose a partition based on a hash of the key 未指定分区但key存在，基于key的hash算法计算得到分区号
 * <li>If no partition or key is present choose the sticky partition that changes when the batch is full. 为指定分区或者key存在选择粘性分区
 * 
 * See KIP-480 for details about sticky partitioning.
 */
public class DefaultPartitioner implements Partitioner {
	
	// 粘性分区行为缓存
    private final StickyPartitionCache stickyPartitionCache = new StickyPartitionCache();

    public void configure(Map<String, ?> configs) {}

    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        return partition(topic, key, keyBytes, value, valueBytes, cluster, cluster.partitionsForTopic(topic).size());
    }
    
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster,
                         int numPartitions) {
        if (keyBytes == null) { // 为指定key从粘性缓存中获取分区号
            return stickyPartitionCache.partition(topic, cluster);
        }
        // hash the keyBytes to choose a partition
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }

    public void close() {}
    
    /**当前粘性分区batch消息以发送完成，变更粘性分区
     * If a batch completed for the current sticky partition, change the sticky partition. 
     * Alternately, if no sticky partition has been determined, set one.
     */
    public void onNewBatch(String topic, Cluster cluster, int prevPartition) {
        stickyPartitionCache.nextPartition(topic, cluster, prevPartition);
    }
}
```

### RoundRobinPartitioner

以轮询的方式将消息均衡的发往各个分区，与key无关。源码如下：

```java
/**
 * The "Round-Robin" partitioner
 * 
 * This partitioning strategy can be used when user wants 
 * to distribute the writes to all partitions equally. This
 * is the behaviour regardless of record key hash. 
 *
 */
public class RoundRobinPartitioner implements Partitioner {
    // 主题对应的分区号
    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap<>();

    public void configure(Map<String, ?> configs) {}

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic); //主题所有分区
        int numPartitions = partitions.size();
        int nextValue = nextValue(topic); //下个分区号
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (!availablePartitions.isEmpty()) { 
            // toPositive code return `number & 0x7fffffff ` 可用分区号的取余
           ` int part = Utils.toPositive(nextValue) % availablePartitions.size();`
            return availablePartitions.get(part).partition();
        } else {
            // no partitions are available, give a non-available partition
            return Utils.toPositive(nextValue) % numPartitions;
        }
    }

    private int nextValue(String topic) {
        AtomicInteger counter = topicCounterMap.computeIfAbsent(topic, k -> {
            return new AtomicInteger(0);
        });
        return counter.getAndIncrement(); // 获取当前分区号后，新增
    }

    public void close() {}

}
```

### UniformStickyPartitioner

batch满时更改粘性分区，与key无关

```java
/** 
 * The partitioning strategy:
 * <ul>
 * <li>If a partition is specified in the record, use it
 * <li>Otherwise choose the sticky partition that changes when the batch is full.
 * 
 * NOTE: In contrast to the DefaultPartitioner, the record key is NOT used as part of the partitioning strategy in this
 *       partitioner. Records with the same key are not guaranteed to be sent to the same partition.  相同的key不保证发往同一个分区
 * 
 * See KIP-480 for details about sticky partitioning.
 */
public class UniformStickyPartitioner implements Partitioner {

    private final StickyPartitionCache stickyPartitionCache = new StickyPartitionCache();

    public void configure(Map<String, ?> configs) {}

    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        return stickyPartitionCache.partition(topic, cluster);
    }

    public void close() {}
    
    public void onNewBatch(String topic, Cluster cluster, int prevPartition) {
        stickyPartitionCache.nextPartition(topic, cluster, prevPartition);
    }
}
```



### StickyPartitionCache

缓存分区行为，用于`DefaultPartitioner`、`StickyPartitionCache`内部。代码如下：

```java
public class StickyPartitionCache {
    // 主题映射分区号
    private final ConcurrentMap<String, Integer> indexCache;
    public StickyPartitionCache() {
        this.indexCache = new ConcurrentHashMap<>();
    }
	// 直接从缓存中读取主题对应的分区号，不存在时获取新的分区号随机获取一个分区号
    public int partition(String topic, Cluster cluster) {
        Integer part = indexCache.get(topic);
        if (part == null) {
            return nextPartition(topic, cluster, -1);
        }
        return part;
    }

    public int nextPartition(String topic, Cluster cluster, int prevPartition) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        Integer oldPart = indexCache.get(topic);
        Integer newPart = oldPart;
        // 检查主题的当前粘着分区是否未设置，或者触发新批处理的分区是否与需要更改的粘着分区相匹配
        // Check that the current sticky partition for the topic is either not set or that the partition that 
        // triggered the new batch matches the sticky partition that needs to be changed.
        if (oldPart == null || oldPart == prevPartition) {
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() < 1) { // 无可用分区
                Integer random = Utils.toPositive(ThreadLocalRandom.current().nextInt());
                newPart = random % partitions.size();
            } else if (availablePartitions.size() == 1) { //可用分区为1
                newPart = availablePartitions.get(0).partition();
            } else {
                // 随机获取分区直到分区号不同
                while (newPart == null || newPart.equals(oldPart)) {
                    int random = Utils.toPositive(ThreadLocalRandom.current().nextInt());
                    newPart = availablePartitions.get(random % availablePartitions.size()).partition();
                }
            }
            // Only change the sticky partition if it is null or prevPartition matches the current sticky partition.   只有在粘着分区为空或prevPartition匹配当前粘着分区时才更改
            if (oldPart == null) {
                indexCache.putIfAbsent(topic, newPart);
            } else {
                indexCache.replace(topic, prevPartition, newPart);
            }
            return indexCache.get(topic);
        }
        return indexCache.get(topic);
    }

}
```



## 自定义分区器

除了使用Kafka提供的分区进行分区外，还可以使用自定义的分区器，只需要实现Partitioner接口，然后再创建KafkaProducer时使用partition.class来显式指定这个分区器即可。代码如下:

```java
public class CustomPartitioner implements Partitioner {
    private AtomicInteger count;

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 打破默认分区的限制：key为null时不会选择非可用分区
        int numPartitions = cluster.partitionsForTopic(topic).size();
        if (Objects.isNull(keyBytes)) {
            return Utils.toPositive(count.getAndIncrement() % numPartitions);
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {
        // 资源的初始化
        count = new AtomicInteger();
    }
}

// 配置
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class.getName());
```











