---
layout: post
title: "Kafka消息序列化"
date: 2022-04-14 00:23:00
categories: Kafka
tags: Kafka
---

* content
{:toc}

在网络中发生数据都是以字节的方式，Kafka也不例外，Producer需要用序列化器(Serializer)把消息转换成字节数组才能通过网络发生给Kafka。Kafka默认提供了多种序列化器，其中常用的serializer如下：

-ByteArraySerializer
-StringSerializer
-ByteBufferSerializer

生产者使用的序列化器和消费者使用的反序列化器要一一对应，否则将导致数据无法解析






## Serializer接口定义

Serializer接口提供了3个方法，代码如下：

````java
/**
 * An interface for converting objects to bytes. 将对象转换为字节数组
 * A class that implements this interface is expected to have a constructor with no parameter. 实现类需包含无参的构造方法
 * @param <T> Type to be serialized from.
 */
public interface Serializer<T> extends Closeable {

    /**
     * Configure this class. 配置当前类
     * @param configs configs in key/value pairs
     * @param isKey whether is for key or value
     */
    default void configure(Map<String, ?> configs, boolean isKey) {
        // intentionally left blank
    }

    /**
     * Convert {@code data} into a byte array.
     *
     * @param topic topic associated with data
     * @param data typed data
     * @return serialized bytes
     */
    byte[] serialize(String topic, T data);

    /**
     * Convert {@code data} into a byte array.
     *
     * @param topic topic associated with data
     * @param headers headers associated with the record
     * @param data typed data
     * @return serialized bytes
     */
    default byte[] serialize(String topic, Headers headers, T data) {
        return serialize(topic, data);
    }

    /** 当前方法可能被KafkaProducer多次调用，需保证此方法的幂等性
     * Close this serializer
     * <p>
     * This method must be idempotent as it may be called multiple times.
     */
    @Override
    default void close() {
        // intentionally left blank
    }
}
````



## 自定义序列化器

如果Kafka提供的序列化器无法满足应用需求，可以实现Serializer接口来自定义自己的序列化器。自定义序列化器需完成下列步骤:

1. 定义数据格式对象

2. 实现Serializer接口，创建自定义序列化器，在serialize方法中实现序列化逻辑

3. 在用于构造KafkaProducer的Properties对象中设置key.serializer、value.serializer

   

   ```java
   public class JacksonSerializer implements Serializer {
       private String encoding = StandardCharsets.UTF_8.name();
   
       private ObjectMapper objectMapper;
   
       @Override
       public void configure(Map configs, boolean isKey) {
           objectMapper = new ObjectMapper();
       }
   
       @Override
       public byte[] serialize(String topic, Object data) {
           try {
               if (data == null)
                   return null;
               else
                   return objectMapper.writeValueAsString(data).getBytes(encoding);
           } catch (Exception e) {
               throw new SerializationException("jackson serializer fail", e);
           }
       }
   }
   ```

   

   ​	配置

   ```java
   prop.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
           prop.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JacksonSerializer.class.getName());
   ```

   



