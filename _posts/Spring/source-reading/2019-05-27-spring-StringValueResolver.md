---
layout: post
title:  "Spring源码-StringValueResolver字符串值解析器接口"
date:   2019-05-27 23:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}


StringValueResolver接口为简单的策略接口，用于解析一个字符串的值，使用在ConfigurableBeanFactory中，JDK1.8后将其定义为函数式接口


```java
/**
 * Simple strategy interface for resolving a String value.
 * Used by {@link org.springframework.beans.factory.config.ConfigurableBeanFactory}. 配置bean工厂
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#resolveAliases 解析别名
 * @see org.springframework.beans.factory.config.BeanDefinitionVisitor#BeanDefinitionVisitor(StringValueResolver) Bean定义访问器
 * @see org.springframework.beans.factory.config.PropertyPlaceholderConfigurer
 */
@FunctionalInterface
public interface StringValueResolver {

	/**
	 * Resolve the given String value, for example parsing placeholders. 解析占位符
	 * @param strVal the original String value (never {@code null})
	 * @return the resolved String value (may be {@code null} when resolved to a null
	 * value), possibly the original String value itself (in case of no placeholders 没有占位符或者忽略不解析占位符
	 * to resolve or when ignoring unresolvable placeholders)
	 * @throws IllegalArgumentException in case of an unresolvable String value
	 */
	@Nullable
	String resolveStringValue(String strVal);

}

```



## EmbeddedValueResolver嵌入式值解析器


```java
/**
 * {@link StringValueResolver} adapter for resolving placeholders and
 * expressions against a {@link ConfigurableBeanFactory}.
 *
 * <p>Note that this adapter resolves expressions as well, in contrast 对比
 * to the {@link ConfigurableBeanFactory#resolveEmbeddedValue} method.
 * The {@link BeanExpressionContext} used is for the plain bean factory, 简单工厂
 * with no scope specified for any contextual objects to access.没有为任何上下文对象指定要访问的范围
 *
 * @author Juergen Hoeller
 * @since 4.3
 * @see ConfigurableBeanFactory#resolveEmbeddedValue(String)
 * @see ConfigurableBeanFactory#getBeanExpressionResolver()
 * @see BeanExpressionContext
 */
public class EmbeddedValueResolver implements StringValueResolver {

	private final BeanExpressionContext exprContext; //bean表达式上下文

	@Nullable
	private final BeanExpressionResolver exprResolver; //Bean表达式解析器


	public EmbeddedValueResolver(ConfigurableBeanFactory beanFactory) {
		this.exprContext = new BeanExpressionContext(beanFactory, null);
		this.exprResolver = beanFactory.getBeanExpressionResolver();
	}


	@Override
	@Nullable
	public String resolveStringValue(String strVal) {
		String value = this.exprContext.getBeanFactory().resolveEmbeddedValue(strVal);
		if (this.exprResolver != null && value != null) {
			Object evaluated = this.exprResolver.evaluate(value, this.exprContext);
			value = (evaluated != null ? evaluated.toString() : null);
		}
		return value;
	}

}
```

- this.exprContext.getBeanFactory().resolveEmbeddedValue(strVal)：

使用ConfigurableBeanFactory接口提供的resolveEmbeddedValue(@Nullable String value)方法进行解析，代码如下：

AbstractBeanFactory#resolveEmbeddedValue(@Nullable String value)
```java
@Override
@Nullable
public String resolveEmbeddedValue(@Nullable String value) {
	if (value == null) {
		return null;
	}
	String result = value;
	for (StringValueResolver resolver : this.embeddedValueResolvers) {
		result = resolver.resolveStringValue(result);
		if (result == null) {
			return null;
		}
	}
	return result;
}
```
通过遍历	List<StringValueResolver\> embeddedValueResolvers = new CopyOnWriteArrayList<\>()对象去器解析，而embeddedValueResolvers解析对象的添加在AbstractApplicationContext#finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)方法中，最终使用`AbstractEnvironment # private final MutablePropertySources propertySources;
ConfigurablePropertyResolver propertyResolver =new PropertySourcesPropertyResolver(this.propertySources);`

PropertySourcesPropertyResolver#resolvePlaceholders()去解析，如下：

AbstractApplicationContext#finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory)

```java
/**
 * Finish the initialization of this context's bean factory,
 * initializing all remaining singleton beans.
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	
	......

	// Register a default embedded value resolver if no bean post-processor 如果没有bean的后置处理器，注册一个默认的嵌入时值解析器
	// (such as a PropertyPlaceholderConfigurer bean) registered any before: 注册前
	// at this point 此时, primarily for resolution in annotation attribute values. 主要用于注释属性值中的解析
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}

	......

}


```

AbstractEnvironment#resolvePlaceholders(String text)

```java
@Override
public String resolvePlaceholders(String text) {
	return this.propertyResolver.resolvePlaceholders(text);
}
```

-	Object evaluated = this.exprResolver.evaluate(value, this.exprContext);

使用Bean表达式解析器解析，简称SpEL,具体使用请查看[Spring中SpEL表达式](/2019/07/03/spring-spel/)


## StringValueResolver接口应用

- EmbeddedValueResolverAware：Spring用于中获取StringValueResolver

```java
@Configuration
@PropertySource(value = "classpath:config/config.properties")
public class PropertiesUtils implements EmbeddedValueResolverAware {


    private StringValueResolver resolver;

    @Override
    public void setEmbeddedValueResolver(StringValueResolver resolver) {
        this.resolver = resolver;
    }

    /**
     *
     * @param key 占位符属性
     * @return
     */
    public String resolver(String key){
        return resolver.resolveStringValue(key);
    }

    public String doExecute(String params){
        return params;
    }

    @Test
    public void test() {
        ApplicationContext applicationContext=new AnnotationConfigApplicationContext(PropertiesUtils.class);

        PropertiesUtils propertiesUtils = applicationContext.getBean(PropertiesUtils.class);

        //获取占位符属性值
        String placeholderName = propertiesUtils.resolver("${custom.config.name}");
        Assert.assertEquals("dlzp",placeholderName);

        //执行bean中的方法，传入参数，获取结果
        String invokingBeanMethod= propertiesUtils.resolver("#{propertiesUtils.doExecute(${custom.config.age})}");
        Assert.assertEquals("30",invokingBeanMethod);


    }
}

```
