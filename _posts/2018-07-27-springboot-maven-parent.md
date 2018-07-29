---
layout: post
title:  "SpringBoot自定义parent"
date:   2018-07-28 17:51:00
categories: SpringBoot 
tags: SpringBoot 
---

* content
{:toc}



[SpringBoot官网](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#using-boot-maven-without-a-parent)
## 继承spring-boot-starter-parent

Maven中SpringBoot项目一般都会将项目配置为继承spring-boot-starter-parent，如下

```xml
<!-继承默认值为Spring Boot - > 

<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.0.BUILD-SNAPSHOT</version>
</parent>

```

然后再POM文件中添加对应的依赖即可：

```xml

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

支持的依赖属性请查看[spring-boot-dependencies pom](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-dependencies/pom.xml)

## 自定义的parent

使用自定义的parent引入spring-boot-starter-parent POM

```xml

<modelVersion>4.0.0</modelVersion>

<groupId>org.dlzp.java</groupId>
<artifactId>dlzp-spring-boot-parent</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>pom</packaging>

<name>dlzp-spring-boot-parent</name>
<url>http://maven.apache.org</url>

<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <spring-boot.version>2.1.0.BUILD-SNAPSHOT</spring-boot.version>
</properties>


<dependencyManagement>
    <dependencies>
		<dependency>
			<!-- Import dependency management from Spring Boot -->
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>2.1.0.BUILD-SNAPSHOT</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

```
`Note`: type 是 pom，scope 是 import，这种类型的 dependency 只能在 dependencyManagement 标签中声明

如果要升级到某个版本，只需要将对应的版本添加到dependencyManagement标签即可：

```java

<dependencyManagement>
	<dependencies>
		<!-- Override Spring Data release train provided by Spring Boot -->
		<dependency>
			<groupId>org.springframework.data</groupId>
			<artifactId>spring-data-releasetrain</artifactId>
			<version>Fowler-SR2</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-dependencies</artifactId>
			<version>2.1.0.BUILD-SNAPSHOT</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>

```


