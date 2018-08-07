---
layout: post
title:  "SpringMvc中@ModelAttribute注解用法"
date:   2018-08-04 23:45:00
categories: SpringMVC
tags: SpringMVC
---

@ModelAttribute注释的方法会在此Controller中的每个方法执行前都会被执行，因此对于一个controller映射多个URL的用法来说，要谨慎使用

* content
{:toc}




## @ModelAttribute注释方法

- @ModelAttribute注释void返回值的方法

在查询列表时如果未提供分页参数时，在进行查询之前动态添加初始信息

```java
public class BaseController {

	/**
	 * 
	 * Description: 添加分頁信息
	 * 
	 * @param page
	 * @param pageNum
	 * @param model
	 */
	@ModelAttribute
	public void initPage(@RequestParam Optional<Integer> page, @RequestParam Optional<Integer> pageNum,
			ModelMap model) {
		if (!page.isPresent())
			model.addAttribute("page", 1);
		else
			model.addAttribute("page", page.get());

		if (!pageNum.isPresent())
			model.addAttribute("pageNum", 10);
		else
			model.addAttribute("pageNum", pageNum.get());

	}
}

@RestController
public class ModelAttributeRest extends BaseController {

	@RequestMapping(value = "/persons")
	public String findPersons(ModelMap model) {  //从ModelMap中获取分页信息
		return model.get("page").toString()+":" + model.get("pageNum").toString();
	}
}


```

- @ModelAttribute注释返回具体类的方法

返回值对象会被默认放到隐含的Model中，在 Model中的 key为 “返回值首字母小写”，value 为返回的值,相当于调用model.addAttribute()

```java
//使用默认提供额key值
@ModelAttribute 
public Person addAccount(@RequestParam String id) { 
  ...
 return person; 
}
``` 
- @ModelAttribute(value="")注释返回具体类的方法

@ModelAttribute注释的value属性，来指定model属性的名称。model属性对象就是方法的返回值

```java
//使用自定义的key值
@ModelAttribute(value = "person"))
public Person addAccount(@RequestParam String id) { 
  ...
 return person; 
}
```
- @ModelAttribute和@RequestMapping同时注释一个方法

```java
@Controller 
public class ModelAttributeRest { 
    @RequestMapping(value = "/helloWorld.do") 
    @ModelAttribute("attributeName") 
    public String helloWorld() { 
         return "hi"; 
      } 
  }
```

返回值并不是表示一个视图名称，而是model属性的值，视图名称由RequestToViewNameTranslator根据请求"/helloWorld.do"转换为逻辑视图helloWorld。Model属性名称有@ModelAttribute(value=””)指定，相当于在request中封装了key=attributeName，value=hi。

## @ModelAttribute注释一个方法的参数

- model中获取值

```java
@Controller 
public class ModelAttributeRest { 
    @ModelAttribute("person") 
    public Person add() { 
        ...
        return person; 
     } 

    @RequestMapping(value = "/person") 
    public Person helloWorld(@ModelAttribute("person") Person person) { 
        person.setxxx(); 
        return person;                   
    } 
}
```

@ModelAttribute("person") Person person注释方法参数，参数person的值来源于add()方法中的model属性,`此时如果方法体没有标注@SessionAttributes("person")，那么scope为request，如果标注了，那么scope为session`

- 从Form表单或URL参数中获取值

@ModelAttribute具有如下三个作用：
    
    1.绑定请求参数到命令对象：处理方法的入参上时，用于将多个请求参数绑定到一个命令对象，从而简化绑定流程，而且自动暴露为模型数据用于视图页面展示时使用。即@ModelAttribute 此处对于供视图页面展示来说与 model.addAttribute("attributeName", xxx); 功能类似


```java

@Controller
public class HelloWorldController { 

    @RequestMapping(value = "/page") 
    public String helloWorld(@ModelAttribute Person person) { //注解@ModelAttribute("person")，作用是将该绑定的命令对象以“person”为名称添加到模型对象中供视图页面展示使用。此时可以在视图页面使用${user.username}来获取绑定的命令对象的属性
        return "page"; 
    } 
}

```

    2.暴露@RequestMapping 方法返回值为模型数据：放在功能处理方法的返回值上时，是暴露功能处理方法的返回值为模型数据，用于视图页面展示时使用。

```java
public @ModelAttribute("person") Person person(@ModelAttribute("person") Person person) //返回值类型是命令对象类型，而且通过 @ModelAttribute("person") 注解，此时会暴露返回值到模型数据（ 名字为person ） 中供视图展示使用;@ModelAttribute  注解的返回值会覆盖 @RequestMapping  注解方法中的 @ModelAttribute  注解的同名命令对象
```
 
    3.暴露表单引用对象为模型数据：放在处理器的一般方法（非功能处理方法）上时，是为表单准备要展示的表单引用对象，如添加预处理信息时，可以在执行功能处理方法（ @RequestMapping  注解的方法）之前，自动添加到模型对象中，用于视图页面展示时使用

 