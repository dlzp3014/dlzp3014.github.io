---
layout: post
title:  "Spring框架中StringUtils#cleanPath与applyRelativePath处理相对文件路径到全路径的表示"
date:   2019-03-28 23:31:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}


需求：将给定的某个全路径+相对路径转换为全路径的表示，如给定"D:/github-workspace/tomcat/ROOT/webapp/application/"路径的相对路径"../../config/config.properties"，获取其相对路径的全路径"D:/github-workspace/tomcat/ROOT/config/config.properties"






```java

public String fullPath(String path, String relativePath) {

    return StringUtils.cleanPath(StringUtils.applyRelativePath(path, relativePath));
}

String relativePath = StringUtils.applyRelativePath("D:/github-workspace/tomcat/ROOT/webapp/application/", "../../config/config.properties");

String fullPath = StringUtils.cleanPath(relativePath);
System.out.println(fullPath); //D:/github-workspace/tomcat/ROOT/config/config.properties


```



