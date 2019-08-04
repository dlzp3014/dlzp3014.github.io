---
layout: post
title:  "Spring源码-ObjectFactory对象工厂接口"
date:   2019-07-29 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Beans
---

* content
{:toc}


ObjectFactory接口用于返回一个对象实例，通常用于封装一个普通的工厂，每次调用返回目标对象的一个实例



## ObjectFactory接口定义


```java
/**
 * Defines a factory which can return an Object instance
 * (possibly shared or independent) when invoked.
 *
 * <p>This interface is typically used to encapsulate a generic factory which
 * returns a new instance (prototype) of some target object on each invocation.
 *
 * <p>This interface is similar to {@link FactoryBean}, but implementations 类似于FactoryBean接口，
 * of the latter are normally meant to be defined as SPI instances in a 后者通常被定义为SPI实例在BeanFactory中
 * {@link BeanFactory}, while implementations of this class are normally meant 当前接口实现通常用来作为API提供给其他bean
 * to be fed as an API to other beans  (through injection). As such, the
 * {@code getObject()} method has different exception handling behavior.
 *
 * @author Colin Sampaleanu
 * @since 1.0.2
 * @see FactoryBean
 */
@FunctionalInterface
public interface ObjectFactory<T> {

	/**
	 * Return an instance (possibly shared or independent 可能共享或独立) 
	 * of the object managed by this factory. 返回由该工厂管理的对象实例
	 * @return the resulting instance
	 * @throws BeansException in case of creation errors
	 */
	T getObject() throws BeansException;

}

```


### RequestObjectFactory:ServletRequest对象工厂

currentRequestAttributes()实现说明参考[Spring源码-WebApplicationContextUtils](/2019/07/18/spring-WebApplicationContextUtils/)

```java
/**
 * Factory that exposes the current request object on demand. 当前请求对象的工厂
 */
@SuppressWarnings("serial")
private static class RequestObjectFactory implements ObjectFactory<ServletRequest>, Serializable {

	@Override
	public ServletRequest getObject() {
		return currentRequestAttributes().getRequest();
	}

	@Override
	public String toString() {
		return "Current HttpServletRequest";
	}
}
```
### ResponseObjectFactory:ServletResponse对象工厂

```java
/**
 * Factory that exposes the current response object on demand. 响应对象
 */
@SuppressWarnings("serial")
private static class ResponseObjectFactory implements ObjectFactory<ServletResponse>, Serializable {

	@Override
	public ServletResponse getObject() {
		ServletResponse response = currentRequestAttributes().getResponse();
		if (response == null) {
			throw new IllegalStateException("Current servlet response not available - " + //当前响应不可用，考虑使用RequestContextFilter代替RequestContextListener
					"consider using RequestContextFilter instead of RequestContextListener");
		}
		return response;
	}

	@Override
	public String toString() {
		return "Current HttpServletResponse";
	}
}
```

### SessionObjectFactory:HttpSession对象工厂

```java
/**
 * Factory that exposes the current session object on demand.
 */
@SuppressWarnings("serial")
private static class SessionObjectFactory implements ObjectFactory<HttpSession>, Serializable {

	@Override
	public HttpSession getObject() {
		return currentRequestAttributes().getRequest().getSession();
	}

	@Override
	public String toString() {
		return "Current HttpSession";
	}
}

```


### WebRequestObjectFactory:WebRequest对象工厂

```java
/**
 * Factory that exposes the current WebRequest object on demand.
 */
@SuppressWarnings("serial")
private static class WebRequestObjectFactory implements ObjectFactory<WebRequest>, Serializable {

	@Override
	public WebRequest getObject() {
		ServletRequestAttributes requestAttr = currentRequestAttributes(); //使用HttpServletRequest和HttpServletResponse创建ServletWebRequest对象
		return new ServletWebRequest(requestAttr.getRequest(), requestAttr.getResponse());
	}

	@Override
	public String toString() {
		return "Current ServletWebRequest";
	}
}

```


### TargetBeanObjectFactory:目标Bean对象工厂

```java
/**
 * Independent inner class - for serialization purposes.
 */
@SuppressWarnings("serial")
private static class TargetBeanObjectFactory implements ObjectFactory<Object>, Serializable {

	private final BeanFactory beanFactory;

	private final String targetBeanName;

	public TargetBeanObjectFactory(BeanFactory beanFactory, String targetBeanName) {
		this.beanFactory = beanFactory;
		this.targetBeanName = targetBeanName;
	}

	@Override
	public Object getObject() throws BeansException {
		return this.beanFactory.getBean(this.targetBeanName);
	}
}
```

## ObjectProvider对象提供者

ObjectProvider接口继承ObjectFactory，允许编程可选操作和宽松非唯一处理，具体实现请参考[Spring源码-DefaultListableBeanFactory默认列表Bean工厂](/2019/07/29/spring-DefaultListableBeanFactory/)


```java
/**
 * A variant  of {@link ObjectFactory} designed specifically for injection points,ObjectFactory多样性，设计为专注注入点
 * allowing for programmatic optionality and lenient not-unique handling.
 *
 * @author Juergen Hoeller
 * @since 4.3
 */
public interface ObjectProvider<T> extends ObjectFactory<T> {

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * <p>Allows for specifying explicit construction arguments, along the 允许指定显式构造参数
	 * lines of {@link BeanFactory#getBean(String, Object...)}.
	 * @param args arguments to use when creating a corresponding instance 创建对应实例时使用的参数
	 * @return an instance of the bean
	 * @throws BeansException in case of creation errors
	 * @see #getObject()
	 */
	T getObject(Object... args) throws BeansException;

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * @return an instance of the bean, or {@code null} if not available 如果没有可用的，返回null
	 * @throws BeansException in case of creation errors
	 * @see #getObject()
	 */
	@Nullable
	T getIfAvailable() throws BeansException;

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * @param defaultSupplier a callback for supplying a default object 回调一个默认对象提供者，如果工厂中不存在时
	 * if none is present in the factory
	 * @return an instance of the bean, or the supplied default object
	 * if no such bean is available
	 * @throws BeansException in case of creation errors
	 * @since 5.0
	 * @see #getIfAvailable()
	 */
	default T getIfAvailable(Supplier<T> defaultSupplier) throws BeansException {
		T dependency = getIfAvailable();
		return (dependency != null ? dependency : defaultSupplier.get());
	}

	/**
	 * Consume an instance (possibly shared or independent) of the object
	 * managed by this factory, if available. 如果可用，消费工厂管理对象的一个实例
	 * @param dependencyConsumer a callback for processing the target object
	 * if available (not called otherwise)
	 * @throws BeansException in case of creation errors
	 * @since 5.0
	 * @see #getIfAvailable()
	 */
	default void ifAvailable(Consumer<T> dependencyConsumer) throws BeansException {
		T dependency = getIfAvailable();
		if (dependency != null) {
			dependencyConsumer.accept(dependency);
		}
	}

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * @return an instance of the bean, or {@code null} if not available or
	 * not unique (i.e. multiple candidates found with none marked as primary) 发现多个候选人，但没有一个被标记为主要候选时抛出异常
	 * @throws BeansException in case of creation errors
	 * @see #getObject()
	 */
	@Nullable
	T getIfUnique() throws BeansException;

	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * @param defaultSupplier a callback for supplying a default object
	 * if no unique candidate is present in the factory
	 * @return an instance of the bean, or the supplied default object
	 * if no such bean is available or if it is not unique in the factory
	 * (i.e. multiple candidates found with none marked as primary)
	 * @throws BeansException in case of creation errors
	 * @since 5.0
	 * @see #getIfUnique()
	 */
	default T getIfUnique(Supplier<T> defaultSupplier) throws BeansException {
		T dependency = getIfUnique();
		return (dependency != null ? dependency : defaultSupplier.get());
	}

	/**
	 * Consume an instance (possibly shared or independent) of the object
	 * managed by this factory, if unique. 如果唯一，消费一个实例
	 * @param dependencyConsumer a callback for processing the target object
	 * if unique (not called otherwise)
	 * @throws BeansException in case of creation errors
	 * @since 5.0
	 * @see #getIfAvailable()
	 */
	default void ifUnique(Consumer<T> dependencyConsumer) throws BeansException {
		T dependency = getIfUnique();
		if (dependency != null) {
			dependencyConsumer.accept(dependency);
		}
	}

}

```



