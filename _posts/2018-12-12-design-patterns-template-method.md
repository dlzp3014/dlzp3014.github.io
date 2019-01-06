---
layout: post
title:  "Template Method 模式：将具体处理过程交给子类实现"
date:   2018-12-01 14:02:00
categories: Design-Patterns
tags: Design-Patterns
---

* content
{:toc}
**Template Method 模式:**是带有模板功能的模式，组成模板的方法被定义在父类中，由于这些方法是抽象方法，所以只查看父类是无法知道这些方法的具体执行过程，唯一能知道的就是父类是如何组合/调用这些方法的;实现这些抽象方法的是子类，在子类中实现了抽象方法也就觉决定了具体的处理过程，即`在不同的子类中实现不同的处理过程，当父类的模板方法被调用时程序的行为也会不同`，但不论子类中的具体实现如何，处理的流程都会按照父类中所定义的那样进行.



## Template Method 模式定义
**在父类中定义处理流程的框架，在子类中实现具体处理**:即定义一个操作中的算法骨架，将一些不走延迟到子类中，使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤

## 使用场景
`在多个子类拥有相同的方法，并且这些方法逻辑相同时，可以考虑使用模板方法模式，在程序的主框架相同、细节不同的场合下，也比较适合使用`
## 示例
把客户端输入的内容存储起来，如redis,数据库，具体步骤为：打开连接，插入数据，关闭连接

- 类图
 
 ![](/img/post.img/diesign-patterns/Template-Method.png)

- 抽象
```java
public abstract class AbstractStorage {

    /**
     * 执行数据插入操作:final禁止子类重写执行过程
     */
    final public void execute(String input) {
        open();
        insert(input);
        close();
    }

    /**
     * 打开连接：protected：不对外提供访问
     */
   protected abstract void open();

    /**
     * 插入数据
     * @param input
     */
    protected abstract void insert(String input);

    /**
     * 关闭连接
     */
    protected abstract void close();
}
```

- 实现

```java
/**
 * 数据库插入
 */
public class DatabaseStorage extends AbstractStorage {

    @Override
    protected void open() {
        System.out.println("open database...");
    }

    @Override
    protected void insert(String input) {
        System.out.println("insert data:" + input);
    }

    @Override
    protected void close() {
        System.out.println("close database...");
    }
}

/**
 * redis插入
 */
public class RedisStorage extends AbstractStorage {

    @Override
    protected void open() {
        System.out.println("open redis...");
    }

    @Override
    protected void insert(String input) {
        System.out.println("insert data:" + input);
    }

    @Override
    protected void close() {
        System.out.println("close redis...");
    }
}
```
- 客户端

```java
public class Client {

    public static void main(String[] args) {

        AbstractStorage storage = null;

        if (args[0].endsWith(":database")) storage = new DatabaseStorage();
        else storage = new RedisStorage();
        storage.execute("input" + args[1]);
    }
}
``` 

## Template Method 模式中的角色
- AbstractClass 抽象类
AbstractClass不仅仅负责实现模板方法，还负责声明在模板方法中所使用到的抽象方法，这些抽象方法有子类ConcreteClas角色实现。
**AbstractClass中的方法分为**：
_基本方法：基本操作，由子类实现的方法，并且在模板方法中被调用_
_模板方法：可以有一个或几个，一般是一个具体方法，也就是一个骨架，实现对基本方法的调用，完成固定的逻辑，为防止恶意的操作，一般模板方法都加上final关键字，不允许被重写_
_钩子方法：由抽象类声明并实现，但子类可以扩展，子类可以通过扩展钩子方法来影响模板方法的逻辑。如可以在AbstractStorage中添加数据校验valid(String input)方法并加以实现_
``抽象类的任务是搭建逻辑的框架，通常由经验丰富的人员编写，因为抽象类的好坏直接决定了程序是否稳定``

- ConcreteClas 具体类
负责具体实现AbstractClass角色中定义的抽象方法，实现的方法将会在AbstractClass中的模板方法中被调用

## Template Method 模式总结

- 易扩展
一般来说，抽象类中的模板方法时不易发生改变的部分，而抽象方法是容易发生变化的部分，因此通过增加实现类，一般可以容易实现功能的扩展，符合开闭原则
- 易维护
对于模板方法模式来说，由于主要逻辑相同，才使用类模板方法，如果不使用模板方法，任由相同的代码散落低分布在不同的类中，则维护起来非常不方便
- 灵活
因为有钩子方法，因此子类的实现也可以影响父类中主要的逻辑运行，但在灵活的同时，由于子类影响到了父类，违反了里氏替换原则，也会给程序带来风险，这就对抽象类的设计有更高的要求

    