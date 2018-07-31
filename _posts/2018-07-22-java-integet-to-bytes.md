---
layout: post
title:  "Java Integer转化为Bytes"
date:   2018-07-28 20:07:00
categories: Java 
tags: Java Integer 
---

* content
{:toc}

一个整数占32位，一个字节占8位，所以使用大小为4个字节的数组来存储整数的每8位




## 先移位再进行再进行与运算

```java
/**
 * 
 * Description:整数转化为字节数组(低位在前, 高位在后)
 * 
 * @param value
 * @return
 */
public static byte[] intToBytes(int value) {
    /**
     * 一个整数占32为，一个字节占8为，所以使用大小为4个字节的属组来存储整数的每8位;
     * &： 按位与，当两位同时为1时才返回1
     */
    byte[] des = new byte[4];
    des[0] = (byte) (value & 0xff);
    des[1] = (byte) ((value >> 8) & 0xff);
    des[2] = (byte) ((value >> 16) & 0xff);
    des[3] = (byte) ((value >> 24) & 0xff);
    return des;
}

/**
 * 
 * Description:byte数组转换成int原始值
 * @param des 
 * @param offset
 * @return
 */
public static int bytesToInt(byte[] des) {
    int value;
      value = (int) (des[offset] & 0xff
                | ((des[offset + 1] & 0xff) << 8)
                | ((des[offset + 2] & 0xff) << 16)
                | (des[offset + 3] & 0xff) << 24);

    return value;
}

```

## 先进行与运算，再移位

```java
public static byte[] intToBytes(int value) { 
    byte[] bytes = new byte[4];
    bytes[3] = (byte) ((value & 0xFF000000)>>24);
    bytes[2] = (byte) ((value & 0x00FF0000)>>16);
    bytes[1] = (byte) ((value & 0x0000FF00)>>8);  
    bytes[0] = (byte) ((value & 0x000000FF));		
    return bytes;
}

public static int bytesToInt(byte[] ary, int offset) {
    int value;	
    value = (int) ((ary[offset]&0xFF) 
            | ((ary[offset+1]<<8) & 0xFF00)
            | ((ary[offset+2]<<16)& 0xFF0000) 
            | ((ary[offset+3]<<24) & 0xFF000000));
    return value;
}

```

