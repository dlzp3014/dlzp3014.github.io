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

- 通过ClassPathResource以类路径的方式进行访问

- 同ServletContextResource以相对于Web应用根目录的方式进行访问

相比于通过JDK的File类访问文件资源的方式，Spring的Resource实现类无疑提供了更加灵活便捷的访问方式，用户可以根据实例情况选择适合的Resource实现类访问资源。下面分别通过FileSystemRessource和ClassPathResource访问同一个文件资源：


```java
Path path = Paths.get(ResourceUtils.class.getClassLoader().getResource("config.properties").toURI());

//使用系统文件路径方式加载文件
WritableResource writableResource = new PathResource(path);

OutputStream outputStream = writableResource.getOutputStream();
outputStream.write("hello word".getBytes());
outputStream.close();

InputStream inputStream = writableResource.getInputStream();

ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
int i;
while ((i = inputStream.read()) != -1) {
    byteArrayOutputStream.write(i);
}
Assert.assertEquals("hello word", byteArrayOutputStream.toString());


//使用类路径方式加载文件
Resource classPathResource = new ClassPathResource("config.properties");

Assert.assertEquals("config.properties",classPathResource.getFilename());


```

获取资源后，就可以通过Resource接口定义的多个方法访问文件的数组和其他信息，如可以通过getFileName()方法获取文件名，通过getFile()方法获取资源对应的File对象，通过getInputStream()方法直接获取文件的输入流；通过WritableResource接口定义的多个方法向文件写入数据，通过getOutputStream()方法直接获取文件的输出流；也可以通过createRelative(String relativePath)在资源相对地址上创建新的文件

在Web应用中，可以通过ServletContextResource以相对于Web应用根目录的方式访问文件资源：

```java
//注意文件资源地址以相对于Web应用根路径的方式表示
ServletContextResource servletContextResource=new ServletContextResourcea(servletContext,"/WEB-INF/classes/config/config.properties");
```

对于位于远程服务器(Web服务器或者FTP服务器)的文件资源，可以通过UrlResource进行访问


资源加载时默认采用系统编码读取资源内容，如果资源文件采用特殊的编码格式，可以通过EncodedResource对资源进行编码，以保证资源内容操作的正确性：

```java
Resource classPathResource = new ClassPathResource("config.properties");

EncodedResource encodedResource = new EncodedResource(classPathResource,"UTF-8");

String content = FileCopyUtils.copyToString(encodedResource.getReader());

```










