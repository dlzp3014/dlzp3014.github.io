---
layout: post
title:  "Spring源码-ApplicationContextInitializer应用上下文初始化接口"
date:   2019-08-04 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Context
---

* content
{:toc}


- ApplicationContextInitializer回调接口，用于在ConfigurableApplicationContext调用refresh()方法之前始化一个Spring的ConfigurableApplicationContext对象

- 通常在web应用程序中使用，需要一些编程方式初始化应用上下文。如注册一个属性源或者active的profiles到上下文环境中(ConfigurableApplicationContext#getEnvironment())。

- ContextLoader和FrameworkServlet支持各自声明一个contextInitializerClasses的context-param和 init-param来指定ApplicationContextInitializer接口的实现

- ApplicationContextInitializer接口实现推荐实现Ordered接口或者标注了Order注解，用于在调用之前对实例进行排序



## ApplicationContextInitializer接口定义

```java
/**
 * Callback interface for initializing a Spring {@link ConfigurableApplicationContext}
 * prior to being {@linkplain ConfigurableApplicationContext#refresh() refreshed}.
 *
 * <p>Typically used within web applications that require some programmatic initialization
 * of the application context. For example, registering property sources or activating
 * profiles against the {@linkplain ConfigurableApplicationContext#getEnvironment()
 * context's environment}. See {@code ContextLoader} and {@code FrameworkServlet} support
 * for declaring a "contextInitializerClasses" context-param and init-param, respectively.
 *
 * <p>{@code ApplicationContextInitializer} processors are encouraged to detect 鼓励检查
 * whether Spring's {@link org.springframework.core.Ordered Ordered} interface has been
 * implemented or if the @{@link org.springframework.core.annotation.Order Order}
 * annotation is present and to sort instances accordingly if so prior to invocation.
 *
 * @author Chris Beams
 * @since 3.1
 * @see org.springframework.web.context.ContextLoader#customizeContext
 * @see org.springframework.web.context.ContextLoader#CONTEXT_INITIALIZER_CLASSES_PARAM
 * @see org.springframework.web.servlet.FrameworkServlet#setContextInitializerClasses
 * @see org.springframework.web.servlet.FrameworkServlet#applyInitializers
 */
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```

## 自定义ApplicationContextInitializer实现

用于加载属性资源

- ApplicationContextInitializer接口实现

```java
public class SysConfig implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext applicationContext) {
        MutablePropertySources propertySources = applicationContext.getEnvironment().getPropertySources();
        Resource resource = new ClassPathResource("config.properties");
        EncodedResource encodedResource = new EncodedResource(resource, Charset.forName("UTF-8"));

        try {
            PropertySource propertySource = new ResourcePropertySource("sysconfig", encodedResource);
            propertySources.addLast(propertySource);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```

配置属性：config.properties

```java
system.config.fileUploadPath=fileUploadPath
```

- web.xml配置

```xml
<context-param>
    <param-name>contextInitializerClasses</param-name>
    <param-value>tech.dlzp.code.spring.mvc.sysconfig.SysConfig</param-value>
</context-param>
```

- 获取配置参数

```java
@Autowired
Environment environment;

@GetMapping("fileUploadPath")
public String getFileUploadPath() {
    return environment.getProperty("system.config.fileUploadPath");
}
```








