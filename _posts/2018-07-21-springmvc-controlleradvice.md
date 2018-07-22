---
layout: post
title:  "SpringMVC中@ControllerAdvice、@RestControllerAdvice注解用法"
date:   2018-07-21 20:04:23
categories: SpringMVC
tags: SpringMVC SpringBoot
---

* content
{:toc}
@ControllerAdvice、@RestControllerAdvice主要用于对控制器的全局配置放置在同一个位置，注解了@Controller类的方法可使用@ExceptionHandler、@InitBinder、@ModelAttribute注解到方法上，这对所有注解了@RequestMapping的控制器内的方法有效。




- @ExceptionHandler:用于全局处理控制器的异常

- @InitBinder:用来设置WebDataBinder,自动绑定前台请求参数到Model中

- @ModelAttribute:让全局的@RequestMapping都能获得此处设置的键值对

## Example:

- @ControllerAdvice 声明一个控制器建言，@ControllerAdvice组合了@Component注解，所以自动注册为Spring Bean
- @ModelAttribute 将键值对添加到全局，所有注解的@RequestMapping的方法可获得次键值对
- @InitBinder 定制WebDataBinder，更多关于WebDataBinder的配置，请参考[WebDataBinder API 文档](https://docs.spring.io/spring/docs/5.0.7.RELEASE/javadoc-api/)

### 定制ControllerAdvice
```java
@ControllerAdvice
public class ExceptionHandlerAdvice {

	@ExceptionHandler(Exception.class)
	public ModelAndView exception(Exception exception, WebRequest request) {
		ModelAndView mav = new ModelAndView("error");
		mav.addObject("errorMsg", exception.getMessage());
		return mav;
	}

	@ModelAttribute
	public void addAttribute(Model model){
		model.addAttribute("msg", "Extra Information");
	}

	@InitBinder //忽略request参数的id
	public void initBinder(WebDataBinder webDataBinder){
		webDataBinder.setDisallowedFields("id");
	}

}

```
### 控制器

```java
@Controller
public class AdviceController {
	@RequestMapping("/advice")
	public String advice(@ModelAttribute("msg") String msg,@RequestParam Integer id) {
		throw new IllegalArgumentException(msg);
	}

}
```

### 异常展示页面

在src/main/resources/views下，创建error.jsp

``` java
<%@page pageEncoding="UTF-8" contentType="text/html; charset=UTF-8"%>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
<title>error</title>
</head>
<body>
    ${errorMsg }
</body>
</html>

```
### 请求地址
/advice?id=1 : advice方法中仅能通过@ModelAttribute的msg信息，而id=null,即过滤掉了id参数，并响应error.jsp页面



