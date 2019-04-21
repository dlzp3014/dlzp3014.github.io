---
layout: post
title:  "Spring中资源抽象接口-Resource"
date:   2019-04-10 23:14:00
categories: Spring 
tags: Spring
---

* content
{:toc}

JDK所一个的访问资源的类(如果java.net.URL、File等)，并不能满足各种底层资源的访问需求，比如缺少从类路径或者WEB容器的上下文中获取资源的操作类。因此，Spring设计了一个Resource接口，它为应用提供了更强大的底层资源访问能力。该接口拥有对应不同资源类型的实现类：






Resource在Spring框架中起着不可或缺的作用，Spring框架使用Resource加载各种资源，包括配置文件资源、国际化属性文件资源等

- WritableResource：可写资源接口，有两个实现类FileSystemResource和PathResource，其中PathResource是Spring 4.x提供的实现类，

- ByteArrayResource:二进制数组表示的资源，二进制数组资源可以在内存中通过程序构造

- ClassPathResource:类路径下的资源，资源以相对于类路径的方式表示

- FileSystemResource:文件系统资源，资源以文件系统路径的方式表示

- InputStreamResource:以输入流返回表示的资源

- ServletContextResource:为访问Web容器上下文中的资源而设计，负责以相对于Web应用根目录的路径加载资源，支持以流和URL的方式访问，在WAR包的情况中，也可以通过File方式访问。该类可以直接从JAR中访问资源

- UrlResource：Url封装了java.net.URL，它使用户能够访问任何可以通过URL表示的资源，如文件系统的资源、HTTP资源、FTP资源

- PathResource:Spring4.X提供的读取资源文件的新类，Path封装了java.net.URL、java.nio.Path(JDK1.7提供)、文件系统资源，它使用用户能够访问任何可以通过URL、Path、系统文件路径表示的资源，如文件系统的资源，HTTP资源，FTP资源

有了Resource抽象的资源类后，就可以将Spring的配置信息放置在任何地方(数据库，LDAP中)，只要最终可以通过Resource接口返回配置信息即可。以上接口的具体实现请查看 [【Spring源码-Resource资源抽象】](/2019/03/21/spring-Resource/)


【Note：】 Spring的Resource接口及其实现类可以在脱离Spring框架的情况下使用，它比通过JDK访问资源的API更好用，更强大


如：读取一个位于Web应用的类路径下文件，可以通过以下访问对这个文件资源进行访问

- 通过FileSystemResource以文件系统绝对路径的方式进行访问

- 











