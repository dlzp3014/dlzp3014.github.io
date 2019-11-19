---
layout: post
title:  "Spring组合注解和元注解"
date:   2018-07-21 20:11:23
categories: Spring 
tags: Spring-Annotation
---

* content
{:toc}

Spring注解主要用来配置注入Bean,切面相关配置(@Transactional)，随着注解的大量使用，尤其是相同的多个注解用到各个类中，显得代码重复度提高，这些就是所谓的模板代码。面对这些问题，Spring中就用到了组合注解来消除这些重复配置，如@configuration就是组合@Component注解，表明这个类其实也是一个Bean。  




所谓元注解其实就是可以注解到其他注解的注解，被注解的注解称之为组合注解，组合组件具备元注解的功能。   

Example:将@configuration和@ComponentScan这个两个元注解组成一个自定义的组合注解 
  
- 自定义组合注解

```java
package com.zp.demo.springboot.config;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
@ComponentScan
public @interface CustomConfigurationAnnotation {
	String[] value() default {};
}

```

- 定义组件

```java
package com.zp.demo.springboot.config;

import org.springframework.stereotype.Component;

@Component
public class DemoComponent {
	
	public String printMessage(String msg) {
		return msg;
	}
	
}

```

- 使用自定义组合注解注解配置类

```java
package com.zp.demo.springboot.config;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;

@CustomConfigurationAnnotation("com.zp.demo.springboot.config")
public class CustomConfiguration {
	
	private static AnnotationConfigApplicationContext context;

	public static void main(String[] args) {
		context = new AnnotationConfigApplicationContext(CustomConfiguration.class);
		DemoComponent component = context.getBean(DemoComponent.class);
		System.out.println(component.printMessage("hello CustomConfig"));
        context.close();
	}
}
```



