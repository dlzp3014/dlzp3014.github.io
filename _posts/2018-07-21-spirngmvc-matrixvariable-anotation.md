---
layout: post
title:  "SpringMVC中@MatrixVariable注解用法"
date:   2018-08-04 22:30:23
categories: SpringMVC 
tags: SpringMVC
---

@MatrixVariable主要用于矩阵变量的绑定参数，拓展了URL请求地址的功能。在Spring MVC中默认是不启用的，启用时需要设置
`<annotation-driven enable-matrix-variables="true" />`;SpringBoot中需要实现WebMvcConfigurer中的configurePathMatch方法




```java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        UrlPathHelper urlPathHelper=new UrlPathHelper();
        urlPathHelper.setRemoveSemicolonContent(false);
        configurer.setUrlPathHelper(urlPathHelper);
    }
}
```

- 映射Get /pets/42;q=11;r=22

```java

@RequestMapping(value = "pets/{petId}, method = RequestMethod.GET)
public void findPet(@PathVariable String petId,@MatrixVariable int q){
    //q将会通过MatrixVariable映射到q上，如果URL中没有;q=11,将会出现异常，需要将其设置为@MatrixVariable（required =false）

}
```

- 映射GET /owners/42;q=11/pets/21;q=22

```java
@RequestMapping(value = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET) 

public void findPet(
    @MatrixVariable(value = "q",pathVar="ownerId") int q1,
    @MatrixVariable(value = "q",pathVar="petId") int q2)
    
    //此处q1=11,q2=22.
}

```

- 映射Get /owners/42;q=11;r=12/pets/21;q=22;s=23

```java
@RequestMapping(value = "/owners/{ownerId}/pets/{petId}", method = RequestMethod.GET)

public void findPet(

     @MatrixVariable Map<String,String> matrixVars,

     @MatrixVariable(pathVar = "petId") Map<String,String> petMatrixVars){

    // matrixVars = ["q" : [11,22], "r" : 12, "s" : 23] 
    // petMatrixVars = ["q" : 11,"s" : 23]
     
...

}
```
**Note:** MatrixVariable中

- 多个变量可以使用“;”（分号）分隔

/cars;color=red;year=2012  

- 一个变量的多个值那么可以使用“,”（逗号）分隔 

color=red,green,blue，如下：

```java
/**
* /find/persons;ids=1,2,3 return [1, 2, 3]
*/
@RestController
public class MatrixVariableRest {

	@GetMapping("/find/{persons}")
	public String findPerson( @MatrixVariable List<Integer> ids) 
	{
		return ids.toString();
	}
}

```
 