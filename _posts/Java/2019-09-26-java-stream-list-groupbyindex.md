---
layout: post
title:  "将大集合List按照索引分割成若干个小集合"
date:   2019-09-25 08:38:00
categories: Java 
tags: Java-Coding
---


使用JDK1.8提供的Stream操作，将集合按照index分割为若干个小的集合，具体实现如下：



```java
/**
 * 按照索引进行分区
 */
public class BigListSplitUtils {

    /**
     * 按照索引进行分区
     *
     * @param indexToInterval 索引对应的区间
     * @param elements        集合列表
     * @param <T>
     * @param <E>
     * @return
     */
    public static <T, E> Map<T, List<E>> groupByIndex(IntFunction<T> indexToInterval, List<E> elements) {
        return IntStream.range(0, elements.size())
                .mapToObj(index -> new ObjectMappingToInterval<>(indexToInterval.apply(index), elements.get(index))) //映射
                .collect(
                        Collectors.groupingBy(item -> item.getInterval(), //按照区间分区
                                Collectors.mapping(item -> item.getElement(), Collectors.toList()) //下级收集器
                        )
                );
    }

    /**
     * @param <T> 区间标识
     * @param <E> 元素
     */
    static class ObjectMappingToInterval<T, E> {

        private T interval;

        private E element;

        public ObjectMappingToInterval(T interval, E element) {
            this.interval = interval;
            this.element = element;
        }

        public T getInterval() {
            return interval;
        }

        public E getElement() {
            return element;
        }
    }
}


```

测试：

```java
List<Integer> integers = IntStream.range(0, 99).boxed().collect(Collectors.toList());

//每10一个区间

Map<Integer, List<Integer>> integerListMap = BigListSplitUtils.groupByIndex(index -> index / 10, integers);

integerListMap.forEach((key, value) -> System.out.println("key:" + key + " -> " + "value:" + value));


```