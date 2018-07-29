---
layout: post
title:  "二分查找"
date:   2018-07-23 22:04:00
categories: Algorithm 
tags: Algorithm 
---

* content
{:toc}

二分查找又称折半查找，它是一种效率较高的查找方法，查找要求`线性表是有序的，且要用向量作为表的存储结构`，在算法家族大类中属于“分治法”，分治法基本都可以用递归来实现的。所有的递归都可以自行定义stack来解递归，所以二分查找法也可以不用递归实现，而且它的非递归实现甚至可以不用栈，因为二分的递归其实是尾递归，它不关心递归前的所有信息。 



  
【注】：因为二分查找需要方便地定位查找区域，所以适合`二分查找的存储结构必须具有随机存储的特性`。因此，该查找方法仅适合于`线性表的顺序存储结构，不适合链式存储结构，且要求元素按关键字有序排列`。如果链表中使用二分查找，其将会变成顺序查找(链表二分查找：将链表排序 -> 定义数组,每个元素的地址依次放入数组-> 通过数组`链表元素地址`来二分查找数据)   
[二分查找文档](https://en.wikipedia.org/wiki/Binary_search_algorithm)



## 原理：

在一个已排序的数组seq中，使用二分查找v，假如这个数组的范围是[low...high]，要查找的v就在这个范围里。查找的方法是拿low到high的正中间的值，假设是m，来跟v相比，如果m>v，说明要查找的v在前数组seq的前半部，否则就在后半部。无论是在前半部还是后半部，将那部分再次折半查找，重复这个过程，知道查找到v值所在的地方。

## 查找过程：
- 给出一个有序的一维数组，并表明想要找到的key，
- 定义两个指针，一个指向数组的首地址left，一个指向数组的最后right，求出数组中间的下标mid=(left+right)/2
- 以mid为划分，如果key=arr[mid]，找到结束；如果key>arr[mid]，在arr[mid]+1到right中间找，并且将left指向arr[mid]+1
，继续在left和right中间根据折半的方法查找，直到找到key结束。如果arr[mid]>key,在left和arr[mid]+1中间找，并且将right指向arr[mid]+1，继续在left和right中间根据折半的方法查，直到找到结束
- 总结 
由边界low、high，求出中间下标mid；比较array[mid]与目标值，由结果缩小范围;找到则退出，未找到重复上面步骤直到 范围size为1且仍不为目标值。

## 复杂度

二分查找的时间复杂度即为while循环的次数，总共有n个元素， 渐渐跟下去就是n,n/2,n/4,….n/2^k，相当为一颗二叉树,其中k就是循环的次数 。 
由于n/2^k取整后>=1， 可得k=log2n,（是以2为底，n的对数），所以时间复杂度可以表示O()=O(logn)

## 优缺点

- 优点:时间复杂度为O(log n)，属于高效的算法
- 缺点:前提条件为数列必须有序。有时候很难保证数组都是有序的，当然可以在构建数组的时候进行排序，可又落到了第二个瓶颈上`数组读取效率是O(1)，可是插入和删除某个元素的效率却是O(n),因而导致构建有序数组变成低效的事情`,解决这些缺陷问题更好的方法应该是使用[二叉查找树](https://en.wikipedia.org/wiki/Binary_search_tree)，最好自然是[平衡二叉查找树](https://en.wikipedia.org/wiki/Self-balancing_binary_search_tree)，既能高效的（O(n log n)）构建有序元素集合，又能如同二分查找法一样快速（O(log n)）的搜寻目标数。

## 二分法查找应用

- 查找有序列表中是否包含target元素，如果存在返回其索引

非递归实现

```java
/**
 * 
 * Description:查找目标数的下标
 * 
 * @param array
 * @param target
 * @return 存在多个返回任意一个即可，没有返回-1
 */
public static int binarySearch(int[] array, int target) {
    int low = 0, high = array.length, mid = -1;
    while (low <= high) {
        /**
         * 如果写成 (low+high)/2时low+high可能会溢出，从而导致数组访问出错
         */
        mid = low + ((high - low) >> 1);

        if (array[mid] > target)// 中值大于目标值，则中值为高位
        {
            high = mid - 1;
        } else if (array[mid] < target)// 中值小于目标值，则中值为低位
        {
            low = mid + 1;
        }
    }
    return mid; // 没找到返回
}

/**
 * 只用小于比较（<）实现二分查找法:减少对目标对象的要求
 */
public static int binarySearch(int[] array, int target) {
    int low = 0, high = array.length;
    int mid = -1;
    while (!(high < low)) {
        mid = (low + high) / 2;
        if (target < array[mid])
            high = mid - 1;
        else if (array[mid] < target)
            low = mid + 1;
        else // find the target
            return mid;
    }
    // the array does not contain the target
    return mid; // 没找到返回
}
```

递归实现

```java
/**
 * 
 * Description:查找目标数的下标 ：递归实现
 * 
 * @param array 
 * @param low
 * @param high
 * @param target
 * @return
 */
public static int binarySearch(int array[], int low, int high, int target) {
    if (low <= high) {
        int mid = (low + high) / 2;

        if (array[mid] > target) {
            return binarySearch(array, low, mid - 1, target);
        } else if (array[mid] < target) {
            return binarySearch(array, mid + 1, high, target);
        } else {
            return mid;
        }
    }
    return -1;
}

public static int binarySearch(int array[], int low, int high, int target) {
    if (low > high)
        return -1;
    int mid = (low + high) / 2;
    if (array[mid] > target)
        return binarySearch(array, low, mid - 1, target);
    if (array[mid] < target)
        return binarySearch(array, mid + 1, high, target);

    return mid;
}
```
- 用二分查找法找寻边界值,也就是说在有序数组中找到“正好大于（小于）目标数”的那个数   
数学表述方式：在集合中找到一个大于（小于）目标数t的数x，使得集合中的任意数要么大于（小于）等于x，要么小于（大于）等于t。如：给予数组和目标数int array = {2, 3, 5, 7, 11, 13, 17};int target = 7;那么上界值应该是11，因为它“刚刚好”大于7；下届值则是5，因为它“刚刚好”小于7   

* 用二分查找法找寻上届:
与精确查找不同之处在于，精确查找分成三类：大于，小于，等于（目标数）。而界限查找则分成了两类：大于和不大于。
如果当前找到的数大于目标数时，它可能就是要找的数，所以需要保留这个索引，因此if (array[mid] > target)时 high=mid; 而没有减1。
```java
//Find the fisrt element, whose value is larger than target, in a sorted array 
public static int binarySearchUpperBound(int array[], int low, int high, int target) {
    if (low > high || target >= array[high]) 
        return -1;

    int mid = (low + high) / 2;
    while (high > low) {
        if (array[mid] > target) //大于目标值时高位赋值为中间值，有可能当前值就是需要找到的值
            high = mid;
        else
            low = mid + 1;  //小于目标值时中间位+1

        mid = (low + high) / 2; //整除，向下取正
    }

    return mid;
}
```

* 用二分查找法找寻下届:

下届寻找基本与上届相同，需要注意的是在取中间索引时，使用了向上取整。若同之前一样使用向下取整，那么当low == high-1，而array[low] 又小于 target时就会形成死循环。因为low无法往上爬超过high。
```java
// Find the last element, whose value is less than target, in a sorted array
public static int binarySearchLowerBound(int array[], int low, int high, int target) {
    // Array is empty or target is less than any every element in array
    if (high < low || target <= array[low])
        return -1;

    int mid = (low + high + 1) / 2; // make mid lean to large side 向上取整
    while (low < high) {
        if (array[mid] < target) 
            low = mid;
        else
            high = mid - 1;

        mid = (low + high + 1) / 2;
    }

    return mid;
}
```
以上两个实现都是找严格界限，也就是要大于或者小于。如果要找松散界限，也就是找到大于等于或者小于等于的值（即包含自身），只要对代码稍作修改就好了，去掉判断数组边界的等号：  

target >= array[high]改为 target > array[high]
在与中间值的比较中加上等号：
array[mid] > target改为array[mid] >= target

* 用二分查找法找寻区域

查找目标数存在的个数，只要找到目标数的严格上届和严格下届，那么界限之间（不包括界限）的数据就是目标数的区域

```java
/**
 * 
 * Description:二分查找目标个数
 * @param array
 * @param target
 * @return
 */
public static int binarySearchCount(int array[], int target) {

    int lower = binarySearchLowerBound(array, 0, array.length - 1, target);

    if (lower + 1 >= array.length || array[lower + 1] != target) {
        return 0;
    }

    int upper = binarySearchUpperBound(array, 0, array.length - 1, target);

    upper = upper < 0 ? (array.length - 1) : (upper - 1);

    return upper - lower;
}
```


