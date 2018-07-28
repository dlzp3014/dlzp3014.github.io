---
layout: post
title:  "Eclipse 创建parent工程"
date:   2018-07-28 00:41:00
categories: Eclipse
tags: Eclipse Maven
---

* content
{:toc}
使用Eclipse创建Maven Project和Maven Module




## 创建Parent Maven Project 
- File --> New --> Other，--> Maven --> Maven Project --> Next

![](/img/post.img/eclipse01.png)

![](/img/post.img/eclipse02.png)

![](/img/post.img/eclipse03.png)

输入Group Id, Artifact Id, Version, Packaging选择pom -> finish 

![](/img/post.img/eclipse04.png)

pom内容：
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>tech.dlzp.maven</groupId>
	<artifactId>maven-parent</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>pom</packaging>
	<name>maven-project</name>
	<modules>
		<module>maven-common</module>
	</modules>
</project>
```

## 创建Maven Module

选择上面创建的Maven Project，右击 -> New Maven Module Project 

![](/img/post.img/eclipse05.png)

![](/img/post.img/eclipse06.png)

![](/img/post.img/eclipse07.png)

pom内容：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>tech.dlzp.maven</groupId>
		<artifactId>maven-parent</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	<artifactId>maven-common</artifactId>
</project>
```

## 创建Maven WEB

![](/img/post.img/eclipse08.png)

![](/img/post.img/eclipse09.png)

![](/img/post.img/eclipse10.png)

![](/img/post.img/eclipse11.png)

## 将Maven 工程转化为WEB工程

如下将已创建的maven Project

![](/img/post.img/eclipse12.png)

project -> Properties -> Project Facets -> Further configuration available...

![](/img/post.img/eclipse13.png)

配置Content directory:"src/main/webapp", 并勾选生成web.xml的选项

![](/img/post.img/eclipse14.png)

配置webapp为项目的根目录,添加Maven依赖 Properties --> Deployment Assembly

![](/img/post.img/eclipse15.png)

最终结构

![](/img/post.img/eclipse16.png)


























