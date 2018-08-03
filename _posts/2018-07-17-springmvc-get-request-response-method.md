---
layout: post
title:  "SpringMVC中获取HttpServletRequest 、HttpServletResponse常用方法"
date:   2018-07-17 22:42:23
categories: SpringMVC 
tags: SpringMVC SpringBoot
---

* content
{:toc}




## Controller方法中添加对于的参数

在Controller方法中添加参数，SpringMVC在处理请求时，会将参数对象对象赋值到方法参数中，[官网文档 1.4.3 Hadnler Methods(Method Arguments)](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-arguments)，获取到参数对象后，如果其他方法中需要使用，就得在这些方法中传入当前对象   
参数对象时方法参数，相当于局部变量，是线程安全的。


```java
@RestController
public class PersonController {

    @RequestMapping("/requestUrl")
	public void findPersonById(HttpServletRequest request){
		
	}
}
```	

## RequestContextHolder获取HttpServletRequest对象

**可以在非Bean中直接获取**

```java

ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder
			.getRequestAttributes();

HttpServletRequest request = requestAttributes.getRequest();

HttpServletResponse response = requestAttributes.getResponse();

String sessionId = requestAttributes.getSessionId();

//ServletContext 上下文
ServletContext context = ContextLoader.getCurrentWebApplicationContext().getServletContext();
```


## @Autowired注解注入

在Spring中，Controller的scope是singleton，整个系统中Controller的实例，但是其中注入的HttpServletRequest却是线程安全的，原因在于 Controller或者Bean初始化时，Spring并没有注入一个request对象，而是注入了一个`代理（proxy）AutowireUtils内部类ObjectFactoryDelegatingInvocationHandler，当调用request的方法时，实际上是调用了由objectFactory.getObject()生成的对象的方法，objectFactory.getObject()生成的对象才是真正的request对象` ,详情查看[@Autowired注入HttpServletRequest`线程安全`源码分析](/2018/07/18/springmvc-source-reading-ObjectFactoryDelegatingInvocationHandler/) 


当Bean中需要使用request对象时,通过该代理获取request对象

```java
@RestController
public class PersonController {
	
	@Autowired 
	private HttpServletRequest request;

	@Autowired 
	private HttpServletResponse response;

	@Autowired
	private HttpSession session;

}
```

## BaseController Autowired注解注入

```java
public class BaseController {
    @Autowired
    protected HttpServletRequest request;     
}

@RestController
public class PersonController extends BaseController{


}

```
