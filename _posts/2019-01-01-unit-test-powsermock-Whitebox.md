---
layout: post
title:  "PowerMock中WhiteBox使用"
date:   2019-01-05 15:04:00
categories: Unit-Test
tags: PowerMock
---

* content
{:toc}

PowerMock 提供WhiteBox的目的就是跳过面向对象语言的封装性，允许test case直接操作类的私有成员、私有方法、甚至通过私有构造函数创建实例。
- 使用Whitebox.setInternalState(..) 设置一个实例或类的非公共成员。
- 使用Whitebox.getInternalState(..) 得到的实例或类的非公共成员。
- 使用Whitebox.invokeMethod(..)调用 实例或类的非公共方法。
- 使用Whitebox.invokeConstructor(..) 私有构造函数创建类的实例。

所有这些方法都可以在不使用PowerMock的情况下实现，这只是普通的Java反射，PowerMock只是提供了这些实用方法





### 获取内部成员值

    boolean result = Whitebox.getInternalState(classToTest, "innerFieldName" );
 
### 设置内部成员值

    Whitebox.setInternalState(classToTest, "innerFieldName", true);  
    
### 直接调用类或者对象的私有方法

    Whitebox.invokeMethod(..)  
    
### 通过私有构造函数创建对象

    Whitebox.invokeConstructor(..)  
    
### 跳过构造函数直接实例化对象
    
    ClassToTest classToTest = Whitebox.newInstance( ClassToTest. class);  
    
### demo
```java
@RunWith(PowerMockRunner.class)
@PrepareForTest({PowserMockDemo.class})

public class UseWhiteBoxTest {

    @Test
    public void testUseWhiteBoxTest() throws Exception {
        PowserMockDemo mockDemo=new PowserMockDemo();
        //获取
        boolean result=Whitebox.getInternalState(mockDemo,"mockPrivateField");
        Assert.assertFalse(result);

        //设置
        Whitebox.setInternalState(mockDemo,"mockPrivateField",true);

        result=Whitebox.getInternalState(mockDemo,"mockPrivateField");
        Assert.assertTrue(result);

        //调用私有方法
        result=Whitebox.invokeMethod(mockDemo,"privateMethod");
        Assert.assertFalse(result);

        //跳过构造函数直接实例化对象
        PowserMockDemo powserMockDemo = Whitebox.newInstance(PowserMockDemo.class);

        Assert.assertFalse(powserMockDemo.mockPrivateMethod());
        //私有构造函数创建对象
        PowserMockDemo demo = Whitebox.invokeConstructor(PowserMockDemo.class);
        Assert.assertFalse(demo.mockPrivateMethod());

    }
}
```