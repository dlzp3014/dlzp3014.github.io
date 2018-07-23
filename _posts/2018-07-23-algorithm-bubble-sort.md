---
layout: post
title:  "冒泡排序"
date:   2018-07-22 21:42:00
categories: Algorithm 
tags: Algorithm Sort-Algorithm
---

* content
{:toc}

冒泡排序是一种计算机科学领域较简单的排序算法，它由经过n-1趟子排序完成，第i趟子排序从第1个数至第n-1个数，若第i个数比后一个数大(则升序，小则降序)则交换两数。`核心思想就是把相邻的元素两两比较，根据大小来交换元素的位置`



## 原理

- 比较相邻元素。如果前面数据大于后面的数据，就交换它们的位置

- 对每一对相邻元素做统一的工作，从开始第一对到结尾的最后一对，最后的元素就是最大值

- 针对所有的元素重复以上步骤，除最后一个外

## 排序过程

假设被排序的数组R[1...n]垂直竖立，将每个数据元素视为有重量的气泡，根据轻气泡不能在重气泡之下的原理，从下往上扫描数组R，凡扫描到违反本原则的轻气泡就使其向上“漂浮”，如此反复进行，直至最后任何两个气泡都是轻者在上，重者在下为止

## 复杂度

冒泡排序是稳定排序，由于该排序算法的每一轮都要遍历所有元素，轮转的次数和元素数量相当，所以时间复杂度是O(N^2)

## 代码实现

- 使用双循环来进行排序，外部循环控制所有的回合，内部循环代表每一轮的冒泡处理，先进行元素比较，在进行元素交换

```java
public static void bubbleSort(int array[]) {
    for (int i = 0; i < array.length; i++) {
        for (int j = 0; j < array.length - i - 1; j++) {
            if (array[j] > array[j + 1]) {
                swap(array, j);
            }
        }
    }
}
```

- 利用布尔变量isSorted作为标记，如果在本轮排序中，元素有交换，则说明数列无序；如果没有元素交换，说明数列已有序，直接跳出大循环

```java
public static void bubbleSort0(int array[]) {
    for (int i = 0; i < array.length; i++) {
        boolean isSorted = true;// 有序标记，每一轮的初始是true
        for (int j = 0; j < array.length - i - 1; j++) {
            if (array[j] > array[j + 1]) {
                swap(array, j);
                // 有元素交换，不是有序，标记为false
                isSorted = false;
            }
        }
        if (isSorted) {
            break;
        }
    }
}
```

- 每轮排序的最后，记录下最后一次元素交换的位置，那个位置也就是无序数列的边界，再完后就是有序区。使用sortBorder标记无序数列的边界，每一轮排序过程中，sortBorder之后的元素就完全不需要比较，肯定是有序的

```java
public static void bubbleSort03(int array[]) {
    int lastExchangeIndex = 0; // 记录最后一次交换的位置
    int sortBorder = array.length - 1;
    for (int i = 0; i < array.length; i++) {
        boolean isSorted = true;// 有序标记，每一轮的初始是true
        for (int j = 0; j < sortBorder; j++) {
            if (array[j] > array[j + 1]) {
                swap(array, j);
                // 有元素交换，不是有序，标记为false
                isSorted = false;
                // 把无序数列的边界更新为最后一次交换元素的位置
                lastExchangeIndex = j;
            }
        }
        sortBorder = lastExchangeIndex;
        if (isSorted) {
            break;
        }
    }
}
```

- 元素交换

```java
//使用临时变量
static void swap(int[] array, int j) {
    int tmp = array[j];
    array[j] = array[j + 1];
    array[j + 1] = tmp;
}

//和差
void swapNotUseTmp(int array[], int i, int j) {
    if (i == j) {
        return;
    }
    array[i] = array[i] + array[j];
    array[j] = array[i] - array[j];
    array[i] = array[i] - array[j];
}
```

- 针对List集合的冒泡排序
   
```java
@SuppressWarnings("unchecked")
public static <T> void bubbleSort(List<? extends Comparable<T>> list) {
    int lastExchangeIndex = 0; // 记录最后一次交换的位置
    int sortBorder = list.size() - 1;
    for (int i = 0; i < list.size(); i++) {
        boolean isSorted = true;// 有序标记，每一轮的初始是true
        for (int j = 0; j < sortBorder; j++) {
            if (list.get(j).compareTo((T) list.get(j + 1)) > 0) {
                swap(list, j);
                // 有元素交换，不是有序，标记为false
                isSorted = false;
                // 把无序数列的边界更新为最后一次交换元素的位置
                lastExchangeIndex = j;
            }
        }
        sortBorder = lastExchangeIndex;
        if (isSorted) {
            break;
        }
    }
}

private static <T> void swap(List<T> list, int j) {
    T tmp = list.get(j);
    list.set(j, list.get(j + 1));
    list.set(j + 1, tmp);
}
```
