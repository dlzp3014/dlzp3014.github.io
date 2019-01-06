---
layout: post
title:  "idea中使用Junit和JUnitGenerator V2.0自动生成测试模块"
date:   2019-01-05 15:40:00
categories: idea Unit-Test
tags: Junit
---
* content
{:toc}







## pom配置

```java
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version>
</dependency>
```

## IDEA配置Junit

打开Settings窗口搜索junit，如图（两个插件都勾选添加）
![](/img/post.img/idea/junit-01.png)

JUnitGenerator V2.0插件，可以帮助我们自动生成测试代码。如果搜索junit没有JUnitGenerator V2.0时，如下图操作（下载添加）：
![](/img/post.img/idea/junit-02.png)
![](/img/post.img/idea/junit-03.png)
![](/img/post.img/idea/junit-04.png)


## 生成测试代码

调用模板的方法（Alt+Insert）默认测试所有所有方法。若想要动态个性化生成，可以在所要测试的类页面上，使用该快捷操作Ctrl + Shift + T,如下图个性化设置：
![](/img/post.img/idea/junit-05.png)
