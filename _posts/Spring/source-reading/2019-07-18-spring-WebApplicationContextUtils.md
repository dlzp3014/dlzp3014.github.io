---
layout: post
title:  "Spring源码-WebApplicationContextUtils"
date:   2019-07-18 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}


WebApplicationContextUtils主要提供了从WebApplicationContext检索ServletContext的方法，对自定义的web 视图或者MVC actions以编程方式访问Spring applicationcontext 非常有用，内容提供的静态方法如下：





- 从ServletContext获取WebApplicationContext对象

ContextLoaderListener会在web应用启动时，将WebApplicationContext加载到ServletContext中，具体实现参考[Spring源码-ContextLoaderListener上下文加载监听器](/2019/07/25/spring-ContextLoaderListener/)

```java
	/**
	 * Find the root {@code WebApplicationContext} for this web app, typically 查找根WebApplicationContext，通过ContextLoaderListener加装
	 * loaded via {@link org.springframework.web.context.ContextLoaderListener}.
	 * <p>Will rethrow an exception that happened on root context startup, 根上下文启动时将重新抛出异常
	 * to differentiate between a failed context startup and no context at all. 区分失败的上下文启动和没有上下文
	 * @param sc ServletContext to find the web application context for 查找web应用程序上下文
	 * @return the root WebApplicationContext for this web app
	 * @throws IllegalStateException if the root WebApplicationContext could not be found
	 * @see org.springframework.web.context.WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE
	 */
	public static WebApplicationContext getRequiredWebApplicationContext(ServletContext sc) throws IllegalStateException {
		WebApplicationContext wac = getWebApplicationContext(sc); //未找到时抛出异常
		if (wac == null) {
			throw new IllegalStateException("No WebApplicationContext found: no ContextLoaderListener registered?");
		}
		return wac;
	}

	/**
	 * Find the root {@code WebApplicationContext} for this web app, typically
	 * loaded via {@link org.springframework.web.context.ContextLoaderListener}.
	 * <p>Will rethrow an exception that happened on root context startup,
	 * to differentiate between a failed context startup and no context at all.
	 * @param sc ServletContext to find the web application context for
	 * @return the root WebApplicationContext for this web app, or {@code null} if none 为找到时返回null
	 * @see org.springframework.web.context.WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE
	 */
	@Nullable
	public static WebApplicationContext getWebApplicationContext(ServletContext sc) {
		return getWebApplicationContext(sc, WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE); // attrName:WebApplicationContext.class.getName() + ".ROOT";
	}

	/**
	 * Find a custom {@code WebApplicationContext} for this web app. 查找自定义的WebApplicationContext
	 * @param sc ServletContext to find the web application context for
	 * @param attrName the name of the ServletContext attribute to look for 要查找的ServletContext属性的名称
	 * @return the desired WebApplicationContext for this web app, or {@code null} if none
	 */
	@Nullable
	public static WebApplicationContext getWebApplicationContext(ServletContext sc, String attrName) {
		Assert.notNull(sc, "ServletContext must not be null");
		Object attr = sc.getAttribute(attrName);
		if (attr == null) {
			return null;
		}
		if (attr instanceof RuntimeException) {
			throw (RuntimeException) attr;
		}
		if (attr instanceof Error) {
			throw (Error) attr;
		}
		if (attr instanceof Exception) {
			throw new IllegalStateException((Exception) attr);
		}
		//未发生任何异常时
		if (!(attr instanceof WebApplicationContext)) { 
			throw new IllegalStateException("Context attribute is not of type WebApplicationContext: " + attr);
		}
		return (WebApplicationContext) attr;
	}

	/**
	 * Find a unique {@code WebApplicationContext} for this web app: either the 查找web应用的唯一WebApplicationContext
	 * root web app context (preferred 优先的) or a unique {@code WebApplicationContext} 
	 * among the registered {@code ServletContext} attributes (typically coming
	 * from a single {@code DispatcherServlet} in the current web application).
	 * <p>Note that {@code DispatcherServlet}'s exposure 暴露 of its context can be DispatcherServlet通过publishContext暴露WebApplicationContext
	 * controlled through its {@code publishContext} property, which is {@code true}
	 * by default but can be selectively switched 有选择地 to only publish a single context
	 * despite  即使 multiple {@code DispatcherServlet} registrations in the web app.
	 * @param sc ServletContext to find the web application context for
	 * @return the desired WebApplicationContext for this web app, or {@code null} if none
	 * @since 4.2
	 * @see #getWebApplicationContext(ServletContext)
	 * @see ServletContext#getAttributeNames()
	 */
	@Nullable
	public static WebApplicationContext findWebApplicationContext(ServletContext sc) {
		WebApplicationContext wac = getWebApplicationContext(sc); //使用默认属性查找，未找到时遍历ServletContext的所有属性值是否为WebApplicationContext，如果是则抛出异常
		if (wac == null) {
			Enumeration<String> attrNames = sc.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = attrNames.nextElement();
				Object attrValue = sc.getAttribute(attrName);
				if (attrValue instanceof WebApplicationContext) {
					if (wac != null) {
						throw new IllegalStateException("No unique WebApplicationContext found: more than one " +
								"DispatcherServlet registered with publishContext=true?");
					}
					wac = (WebApplicationContext) attrValue;
				}
			}
		}
		return wac;
	}
```


- BeanFactory注册WebApplication的作用域

包括：request，session，globalSession、application

```java
	/**
	 * Register web-specific scopes ("request", "session", "globalSession")
	 * with the given BeanFactory, as used by the WebApplicationContext.
	 * @param beanFactory the BeanFactory to configure
	 */
	public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory) {
		registerWebApplicationScopes(beanFactory, null);
	}

	/**
	 * Register web-specific scopes ("request", "session", "globalSession", "application")
	 * with the given BeanFactory, as used by the WebApplicationContext.
	 * @param beanFactory the BeanFactory to configure
	 * @param sc the ServletContext that we're running within
	 */
	public static void registerWebApplicationScopes(ConfigurableListableBeanFactory beanFactory,
			@Nullable ServletContext sc) {

		beanFactory.registerScope(WebApplicationContext.SCOPE_REQUEST, new RequestScope()); //请求
		beanFactory.registerScope(WebApplicationContext.SCOPE_SESSION, new SessionScope());// session
		if (sc != null) {
			ServletContextScope appScope = new ServletContextScope(sc); 
			beanFactory.registerScope(WebApplicationContext.SCOPE_APPLICATION, appScope); //application
			// Register as ServletContext attribute, for ContextCleanupListener to detect it. 注册为ServletContext属性，以便ContextCleanupListener检测它
			sc.setAttribute(ServletContextScope.class.getName(), appScope);
		}
		
		注册具有相应自动获取值的特殊依赖项类型：ServletRequest注册到 bean工厂中，使用对象工厂生成具体的对象值

		beanFactory.registerResolvableDependency(ServletRequest.class, new RequestObjectFactory());
		beanFactory.registerResolvableDependency(ServletResponse.class, new ResponseObjectFactory());
		beanFactory.registerResolvableDependency(HttpSession.class, new SessionObjectFactory());
		beanFactory.registerResolvableDependency(WebRequest.class, new WebRequestObjectFactory());
		if (jsfPresent) {
			FacesDependencyRegistrar.registerFacesDependencies(beanFactory);
		}
	}

```

- 注册EnvironmentBean

```java
	/**
	 * Register web-specific environment beans ("contextParameters", "contextAttributes") 
	 * with the given BeanFactory, as used by the WebApplicationContext.
	 * @param bf the BeanFactory to configure
	 * @param sc the ServletContext that we're running within
	 */
	public static void registerEnvironmentBeans(ConfigurableListableBeanFactory bf, @Nullable ServletContext sc) {
		registerEnvironmentBeans(bf, sc, null);
	}

	/**
	 * Register web-specific environment beans ("contextParameters", "contextAttributes")
	 * with the given BeanFactory, as used by the WebApplicationContext.
	 * @param bf the BeanFactory to configure
	 * @param servletContext the ServletContext that we're running within
	 * @param servletConfig the ServletConfig of the containing Portlet
	 */
	public static void registerEnvironmentBeans(ConfigurableListableBeanFactory bf,
			@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
 		//bean工厂中不包含ServletContext，注册其为单例bean
		if (servletContext != null && !bf.containsBean(WebApplicationContext.SERVLET_CONTEXT_BEAN_NAME)) { servletContext
			bf.registerSingleton(WebApplicationContext.SERVLET_CONTEXT_BEAN_NAME, servletContext);
		}
		//servletConfig
		if (servletConfig != null && !bf.containsBean(ConfigurableWebApplicationContext.SERVLET_CONFIG_BEAN_NAME)) { 
			bf.registerSingleton(ConfigurableWebApplicationContext.SERVLET_CONFIG_BEAN_NAME, servletConfig);
		}

		bean工厂中无contextParameters时，将servletContext和servletConfig中的参数封装到Map中，注册到bean工厂中

		if (!bf.containsBean(WebApplicationContext.CONTEXT_PARAMETERS_BEAN_NAME)) { contextParameters
			Map<String, String> parameterMap = new HashMap<>();
			if (servletContext != null) {
				Enumeration<?> paramNameEnum = servletContext.getInitParameterNames();
				while (paramNameEnum.hasMoreElements()) {
					String paramName = (String) paramNameEnum.nextElement();
					parameterMap.put(paramName, servletContext.getInitParameter(paramName));
				}
			}
			if (servletConfig != null) {
				Enumeration<?> paramNameEnum = servletConfig.getInitParameterNames();
				while (paramNameEnum.hasMoreElements()) {
					String paramName = (String) paramNameEnum.nextElement();
					parameterMap.put(paramName, servletConfig.getInitParameter(paramName));
				}
			}
			bf.registerSingleton(WebApplicationContext.CONTEXT_PARAMETERS_BEAN_NAME,
					Collections.unmodifiableMap(parameterMap));
		}

		if (!bf.containsBean(WebApplicationContext.CONTEXT_ATTRIBUTES_BEAN_NAME)) { contextAttributes
			Map<String, Object> attributeMap = new HashMap<>();
			if (servletContext != null) {
				Enumeration<?> attrNameEnum = servletContext.getAttributeNames();
				while (attrNameEnum.hasMoreElements()) {
					String attrName = (String) attrNameEnum.nextElement();
					attributeMap.put(attrName, servletContext.getAttribute(attrName));
				}
			}
			bf.registerSingleton(WebApplicationContext.CONTEXT_ATTRIBUTES_BEAN_NAME,
					Collections.unmodifiableMap(attributeMap));
		}
	}
```


- 初始化Servlet属性源：

在创建StandardServletEnvironment(AbstractEnvironment)对象时会调用#customizePropertySources()方法，使用StubPropertySource对象来代替真实的ServletContext、ServletConfig对象中会，以保证其属性源具有最有权，随后在initPropertySources中进行初始化，原因在于在创建application context时，ServletContext对象在ApplicationContext内部不可用，StubPropertySource文档说明如下：

``` java
@code PropertySource} to be used as a placeholder in cases where an actual
 property source cannot be eagerly initialized at application context
 creation time.  For example, a {@code ServletContext}-based property source
 must wait until the {@code ServletContext} object is available to its enclosing
 {@code ApplicationContext}.  In such cases, a stub should be used to hold the
 intended default position/order of the property source, then be replaced
 during context refresh.
```



```java
	/**
	 * Convenient variant of {@link #initServletPropertySources(MutablePropertySources,
	 * ServletContext, ServletConfig)} that always provides {@code null} for the
	 * {@link ServletConfig} parameter.
	 * @see #initServletPropertySources(MutablePropertySources, ServletContext, ServletConfig)
	 */
	public static void initServletPropertySources(MutablePropertySources propertySources, ServletContext servletContext) {
		initServletPropertySources(propertySources, servletContext, null);
	}

 	替换基于Servlet的StubPropertySource属性源(占位属性源)使用真实的实例填充给定的servletContext和servletConfig对象

	/**
	 * Replace {@code Servlet}-based {@link StubPropertySource stub property sources} with
	 * actual instances populated with the given {@code servletContext} and
	 * {@code servletConfig} objects.
	 * <p>This method is idempotent 幂等 with respect方面 to the fact事实 这种方法对事实是幂等的 it may be called any number 在任何时间被多次调用，但是将执行替代子属性源
	 * of times but will perform replacement of stub property sources with their
	 * corresponding 相应 actual property sources once and only once. 一次且仅一次
	 * @param sources the {@link MutablePropertySources} to initialize (must not 初始化的MutablePropertySources
	 * be {@code null})
	 * @param servletContext the current {@link ServletContext} (ignored if {@code null}
	 * or if the {@link StandardServletEnvironment#SERVLET_CONTEXT_PROPERTY_SOURCE_NAME
	 * servlet context property source} has already been initialized)
	 * @param servletConfig the current {@link ServletConfig} (ignored if {@code null}
	 * or if the {@link StandardServletEnvironment#SERVLET_CONFIG_PROPERTY_SOURCE_NAME
	 * servlet config property source} has already been initialized)
	 * @see org.springframework.core.env.PropertySource.StubPropertySource
	 * @see org.springframework.core.env.ConfigurableEnvironment#getPropertySources()
	 */
	public static void initServletPropertySources(MutablePropertySources sources,
			@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {

		Assert.notNull(sources, "'propertySources' must not be null");
		String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME; //servletContextInitParams

		存在且为StubPropertySource实例替换为ServletContextPropertySource对象
		if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
			sources.replace(name, new ServletContextPropertySource(name, servletContext));
		}
		
		name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME; //servletConfigInitParams
		if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
			sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
		}
	}
```


- 获取当前RequestAttributes实例作为ServletRequestAttributes

当前方法为私有静态，提供给RequestObjectFactory、ResponseObjectFactory、SessionObjectFactory、WebRequestObjectFactory中获取对象的泛型参数。详细说明请查看[Spring源码-ObjectFactory对象工厂接口](/2019/07/29/spring-ObjectFactory/)


RequestContextHolder#currentRequestAttributes()用于获取当前请求的属性RequestAttributes(ServletRequestAttributes)，可以仿写该私有静态方法在任何地方获取当前线程的HttpServletRequest，HttpServletResponse，HttpSession对象。具体实现查看[Spring源码-RequestContextHolder请求上下文持有者](/2019/07/29/spring-RequestContextHolder/)


```java
/**
 * Return the current RequestAttributes instance as ServletRequestAttributes. 返回
 * @see RequestContextHolder#currentRequestAttributes()
 */
private static ServletRequestAttributes currentRequestAttributes() {
	RequestAttributes requestAttr = RequestContextHolder.currentRequestAttributes();
	if (!(requestAttr instanceof ServletRequestAttributes)) {
		throw new IllegalStateException("Current request is not a servlet request");
	}
	return (ServletRequestAttributes) requestAttr;
}

```