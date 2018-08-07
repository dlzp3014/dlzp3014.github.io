---
layout: post
title:  "SpringMVC中使用@CrossOrigin解决跨越问题"
date:   2018-07-21 20:11:23
categories: SpringMVC 
tags: SpringMVC
---

* content
{:toc}

出于安全原因，浏览器禁止Ajax调用驻留在当前原点之外的资源,即阻止跨域的ajax请求。具体而言，Web 应用程序能且只能使用 XMLHttpRequest 对象向其加载的源域名发起 HTTP 请求，而不能向任何其它域名发起请求。
   
Spring MVC 从4.2版本开始增加了对CORS的支持,[官方文档](https://docs.spring.io/spring/docs/5.0.8.RELEASE/spring-framework-reference/web.html#mvc-cors-controller)

## 某个方法支持跨域访问

Controller中配置CORS，可以向@RequestMapping注解处理程序方法添加一个@CrossOrigin注解，以便启用CORS（默认情况下，@CrossOrigin允许在@RequestMapping注解中指定的所有源和HTTP方法）

```java
@RestController
public class CorsRest {
	
	@CrossOrigin
	@GetMapping("/{id}")
	public void cros(@PathVariable Long id) {
		
	}
}
```

## 使用HttpServletResponse设置跨域请求头信息

```java
@RequestMapping
public Object index(HttpServletRequest request, HttpServletResponse response, @CookieValue(value = "sid", required = false) String sid) {
	response.setHeader("Access-Control-Allow-Origin","http://a.test.com"); //允许跨域的Origin设置
	response.setHeader("Access-Control-Allow-Credentials","true"); //允许携带cookie
	logger.info("cookie sid = " + sid);
	return restTemplateService.someRestCall();
}
```


## 整个controller支持跨域访问

```java

@CrossOrigin(origins = "http://domain2.com", maxAge = 3600)
@RestController
public class CorsRest {
	
	@GetMapping("/{id}")
	public void cros(@PathVariable Long id) {
		
	}
}

```

## Spring Security中启用CORS
[Spring Security 文档](https://docs.spring.io/spring-security/site/docs/current/reference/html/cors.html)
```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors().and()...
    }
}
```

## 全局CORS配置

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {

        registry.addMapping("/api/**")
            .allowedOrigins("http://domain2.com")
            .allowedMethods("PUT", "DELETE")
            .allowedHeaders("header1", "header2", "header3")
            .exposedHeaders("header1", "header2")
            .allowCredentials(true).maxAge(3600);

        // Add more mappings...
    }
}
```

## 基于过滤器的CORS

```java

@Bean
public FilterRegistrationBean corsFilter() {
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowCredentials(true);
    config.addAllowedOrigin("http://domain1.com");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");
    source.registerCorsConfiguration("/**", config);
    FilterRegistrationBean bean = new FilterRegistrationBean(new CorsFilter(source));
    bean.setOrder(0);
    return bean;
}

```

## @CrossOrigin工作机制

* CORS请求被自动发送到注册的各种HandlerMapping,它们处理CORS准备请求并拦截CORS简单和实际请求，这得益于CorsProcessor实现（默认情况下默认DefaultCorsProcessor处理器），以便添加相关的CORS响应头（如Access-Control-Allow-Origin）。 CorsConfiguration 允许您指定CORS请求应该如何处理：允许origins, headers, methods等。
* AbstractHandlerMapping#setCorsConfiguration() 允许指定一个映射，其中有几个CorsConfiguration 映射在路径模式上，比如/api/**。
* 子类可以通过重写AbstractHandlerMapping类的getCorsConfiguration(Object, HttpServletRequest)方法来提供自己的CorsConfiguration。
* 处理程序可以实现 CorsConfigurationSource接口（如ResourceHttpRequestHandler），以便为每个请求提供一个CorsConfiguration。

* Note：服务器端 Access-Control-Allow-Credentials = true时，参数Access-Control-Allow-Origin 的值不能为 '*'，必须为具体的origin

## 前端调用方式

[参考文档](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)

- 原生ajax调用示例：

```javascript
var xhr = new XMLHttpRequest();  
xhr.open("POST", "http://b.test.com/api/rest", true);  
xhr.withCredentials = true; //支持跨域发送cookies
xhr.send();

```
- jQuery调用示例：

```javascript
$.ajax({
    url: 'http://b.test.com/api/rest',
    dataType: 'json',
    type : 'POST',
    xhrFields: {
        withCredentials: true //是否携带cookie
    },
    crossDomain: true,
    contentType: "application/json",
    success: (res) => {
      console.log(res);
    }
  });
  
```  
- fetch方式

```javascript
fetch('http://b.test.com/api/rest', 
  {credentials: 'include'}  //注意这里的设置，支持跨域发送cookies
).then(function(res) {
  if (res.ok) {
    res.json().then(function(data) {
      console.log(data.value);
    });
  } else {
    console.log("Looks like the response wasn't perfect, got status", res.status);
  }
}, function(e) {
  console.log("Fetch failed!", e);
});

```

