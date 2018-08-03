---
layout: post
title:  "Java按位操作应用"
date:   2018-08-01 23:46:00
categories: Java
tags: Java
---

* content
{:toc}




## 按位与运算

- 清零操作
将二进制全部位都值为零

```java
x & 0
```

- 取指定位数的值

```java
x & 0000 1111 取低四位
n & 0x0000FFFF 取低位
n & 0xFFFF0000 取高位
```

- 奇偶判断

```java
n & 1，等于0为偶，等于1为奇
```
- 取余

```java
n % m , 如m为2的幂次方，可用(n & (m - 1))
```

- x & ( x - 1 )

将二进制数最右边的1变成0，如果没有1则生成的二进制位全为0,用于检查一个无符号数是否是2的幂

```java
x = 1010 1000
x-1 = 1010 0111
& = 1010 0000 

```
- x & ( x + 1 )

将二进制数最右边0后的1全置为0。可以用来检测一个无符号整数是否为2^n - 1的形式(包括0和所有位均为1的情况)
```java
x = 1010 1000
x+1 = 1010 1001
& = 1010 1000 

```

- x & ( -x ) 

解析出最右侧1的位

```java
x = 0000 0000 0000 0000 0000 0000 0101 1000
-x= 1111 1111 1111 1111 1111 1111 1010 1000
x&(-x) = 00001000

```

- 统计二进制中1的个数

```java
/**
 * Description: 计算整数二进制中1的个数
 * 
 * @param x
 * @return
 */
public static int countNumeOne(int x) {
    int countx = 0;
    do {
        countx++;
        x = x & (x - 1);
    } while (x != 0);
    return countx;
}
```


## 按位或运算

- 将某些位上的数值为1

```java
x|0000 1111 将低四位上的数值为1
```

## 按位异或运算

- 翻转指定位上的值

```java
x^0000 1111 翻转低四位

```

- 异或自反性

x^0 与0异或保留原值; x ^ x = 0

```java
/**
 * 假定有2K+1个数，其中有2k个相同，需要找出不相同的那个数
 * Description:查找不同
 */
public static int findDif(int[] array) {

    int result = 0;

    for (int i = 0; i < array.length; i++) {

        result ^= array[i];
        if (result != 0) {
            return result;
        }
    }
    return result;
}
```
- 交换两个变量的值

```java
/**
 * 
 * Description:交换两个数
 */
public static void swap(int a, int b) {
    a = a ^ b;
    b = a ^ b;
    a = a ^ b;
}
```

## 按位取反

- 将一个数的最低位值为0

```java
x|~1
```

- ( ~x ) & ( x + 1 )

解析最右侧的0位

```java
x = 0000 0000 0000 0000 0000 0000 0101 0111
~x= 1111 1111 1111 1111 1111 1111 1010 1000 
x+1=0000 0000 0000 0000 0000 0000 0101 1000
& = 0000 1000 

```

## 左移运算

- 乘法

```java
n * 2 等价于 n << 1
n * 5 等价于 n << 2 + 1
```
## 右移运算

- 除法

```java
n / 2 等价于 n >> 1
```
- 正负判断

```java
(n >>> 31) & 1，等于0为正，等于1为负
```

- 最大值

```java
(1 << 31) – 1 = 0x7FFFFFFF
```
- 最小值

```java
1 << 31 = 0x80000000
```



