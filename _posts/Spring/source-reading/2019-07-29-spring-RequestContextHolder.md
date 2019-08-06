---
layout: post
title:  "Spring源码-RequestContextHolder请求上下文持有者"
date:   2019-07-29 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Web
---

* content
{:toc}

RequestContextHolder类用于以线程绑定的RequestAttributes对象的形式暴露web请求

- 如果inheritable标记为true时，由当前线程派生的任何子线程将被继承这个请求

- 使用RequestContextListener请求上下文监听器或者RequestContextFilter请求上下文将暴露当前web 请求

- DispatcherServlet默认已经暴露当前请求(FrameworkServlet#initContextHolders()方法中进行)


```java
// processRequest() -> initContextHolders() -> doService()
private void initContextHolders(HttpServletRequest request,
		@Nullable LocaleContext localeContext, @Nullable RequestAttributes requestAttributes) {

	...
	if (requestAttributes != null) {
		RequestContextHolder.setRequestAttributes(requestAttributes, this.threadContextInheritable);
	...
}
```





## 成员变量

通过使用ThreadLocal和InheritableThreadLocal来实现RequestAttributes对象的线程安全问题。ThreadLocal详细说明查看[JDK1.8源码-ThreadLocal线程局部变量](/2019/08/06/jdk1.8-source-reading-ThreadLocal/)

```java
/**
 * Holder class to expose the web request in the form of a thread-bound 线程绑定
 * {@link RequestAttributes} object. The request will be inherited
 * by any child threads spawned by the current thread if the
 * {@code inheritable} flag is set to {@code true}.
 *
 * <p>Use {@link RequestContextListener} or
 * {@link org.springframework.web.filter.RequestContextFilter} to expose
 * the current web request. Note that
 * {@link org.springframework.web.servlet.DispatcherServlet}
 * already exposes the current request by default.
 *
 * @author Juergen Hoeller
 * @author Rod Johnson
 * @since 2.0
 * @see RequestContextListener
 * @see org.springframework.web.filter.RequestContextFilter
 * @see org.springframework.web.servlet.DispatcherServlet
 */
public abstract class RequestContextHolder  {

	private static final boolean jsfPresent =
			ClassUtils.isPresent("javax.faces.context.FacesContext", RequestContextHolder.class.getClassLoader());

	private static final ThreadLocal<RequestAttributes> requestAttributesHolder =
			new NamedThreadLocal<>("Request attributes"); //当前线程

	private static final ThreadLocal<RequestAttributes> inheritableRequestAttributesHolder =
			new NamedInheritableThreadLocal<>("Request context"); //继承

```

## 核心静态方法

- 重置RequestAttributes

```java
/**
 * Reset the RequestAttributes for the current thread.
 */
public static void resetRequestAttributes() {
	requestAttributesHolder.remove();
	inheritableRequestAttributesHolder.remove();
}
```

- 绑定RequestAttributes到当前线程中

```java

	/**
	 * Bind the given RequestAttributes to the current thread,
	 * <i>not</i> exposing it as inheritable for child threads. 子线程不继承
	 * @param attributes the RequestAttributes to expose
	 * @see #setRequestAttributes(RequestAttributes, boolean)
	 */
	public static void setRequestAttributes(@Nullable RequestAttributes attributes) {
		setRequestAttributes(attributes, false);
	}

	/**
	 * Bind the given RequestAttributes to the current thread.
	 * @param attributes the RequestAttributes to expose,
	 * or {@code null} to reset the thread-bound context
	 * @param inheritable whether to expose the RequestAttributes as inheritable
	 * for child threads (using an {@link InheritableThreadLocal})
	 */
	public static void setRequestAttributes(@Nullable RequestAttributes attributes, boolean inheritable) {
		if (attributes == null) { //null时清除
			resetRequestAttributes();
		}
		else {
			if (inheritable) { //继承
				inheritableRequestAttributesHolder.set(attributes);//设置子线程，移除父线程
				requestAttributesHolder.remove();
			}
			else {
				requestAttributesHolder.set(attributes); //设置父线程，移除子线程
				inheritableRequestAttributesHolder.remove();
			}
		}
	}

```

- 获取当前线程绑定的RequestAttributes对象

```java
	/**
	 * Return the RequestAttributes currently bound to the thread.
	 * @return the RequestAttributes currently bound to the thread,
	 * or {@code null} if none bound 没找到时返回null
	 */
	@Nullable
	public static RequestAttributes getRequestAttributes() {
		RequestAttributes attributes = requestAttributesHolder.get(); //父线程
		if (attributes == null) {
			attributes = inheritableRequestAttributesHolder.get(); //子线程
		}
		return attributes;
	}

	/**
	 * Return the RequestAttributes currently bound to the thread.
	 * <p>Exposes the previously bound RequestAttributes instance, if any. 暴露以前绑定的RequestAttributes实例，为null时抛出异常
	 * Falls back to the current JSF FacesContext, if any.
	 * @return the RequestAttributes currently bound to the thread
	 * @throws IllegalStateException if no RequestAttributes object
	 * is bound to the current thread
	 * @see #setRequestAttributes
	 * @see ServletRequestAttributes
	 * @see FacesRequestAttributes
	 * @see javax.faces.context.FacesContext#getCurrentInstance()
	 */
	public static RequestAttributes currentRequestAttributes() throws IllegalStateException {
		RequestAttributes attributes = getRequestAttributes();
		if (attributes == null) {
			if (jsfPresent) {
				attributes = FacesRequestAttributesFactory.getFacesRequestAttributes();
			}
			if (attributes == null) {
				throw new IllegalStateException("No thread-bound request found: " +
						"Are you referring to request attributes outside of an actual web request, " +
						"or processing a request outside of the originally receiving thread? " +
						"If you are actually operating within a web request and still receive this message, " +
						"your code is probably running outside of DispatcherServlet/DispatcherPortlet: " +
						"In this case, use RequestContextListener or RequestContextFilter to expose the current request.");
			}
		}
		return attributes;
	}


	/**
	 * Inner class to avoid hard-coded JSF dependency.
 	 */
	private static class FacesRequestAttributesFactory {

		@Nullable
		public static RequestAttributes getFacesRequestAttributes() {
			FacesContext facesContext = FacesContext.getCurrentInstance();
			return (facesContext != null ? new FacesRequestAttributes(facesContext) : null);
		}
	}

}
```




