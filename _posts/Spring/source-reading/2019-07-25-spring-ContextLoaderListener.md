---
layout: post
title:  "Spring源码-ContextLoaderListener上下文加载监听器"
date:   2019-07-25 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Web
---

* content
{:toc}


ContextLoaderListener上下文加载监听器用于引导监听WebApplicationContext的启动和关闭，从Spring3.1开始，ContextLoaderListener支持通过ContextLoaderListener(WebApplicationContext)构造函数注入，在Servlet3.0+环境中允许编程配置，类结构如下：

![](/img/post.img/spring/ContextLoaderListener.png)


web.xml 常用配置如下：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>
        classpath:spring-config.xml <!--可以使用通配符，如classpath:spring-*.xml ，默认加载路径为/WEB-INF/applicationContext.xml-->
    </param-value>
</context-param>
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>

```






## ContextLoaderListener


- 类说明

```java
/**
 * Bootstrap listener to start up and shut down Spring's root {@link WebApplicationContext}.
 * Simply delegates to {@link ContextLoader} as well as to {@link ContextCleanupListener}.
 * 简单委托给ContextLoader和ContextCleanupListener
 * <p>As of Spring 3.1, {@code ContextLoaderListener} supports injecting the root web
 * application context via the {@link #ContextLoaderListener(WebApplicationContext)}
 * constructor, allowing for programmatic configuration in Servlet 3.0+ environments.
 * See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
 *
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 17.02.2003
 * @see #setContextInitializers
 * @see org.springframework.web.WebApplicationInitializer
 */
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {

}
```

- 构造函数



```java
/**
 * Create a new {@code ContextLoaderListener} that will create a web application
 * context based on the "contextClass" and "contextConfigLocation" servlet 基于contextClass和contextConfigLocation上下文参数创建web应用上下文
 * context-params. See {@link ContextLoader} superclass documentation for details on
 * default values for each.
 * <p>This constructor is typically used when declaring {@code ContextLoaderListener}
 * as a {@code <listener>} within {@code web.xml}, where a no-arg constructor is
 * required. 当在web.xml中声明一个ContextLoaderListener时(无参数构造方法要求),通常使用这个构造函数
 * <p>The created application context will be registered into the ServletContext under
 * the attribute name {@link WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE}
 * and the Spring application context will be closed when the {@link #contextDestroyed}
 * lifecycle method is invoked on this listener. 创建的application context将被注入到ServletContext的属性名中，当Spring的应用上下文关闭时，contextDestroyed声明周期方法在当前监听器中执行
 * @see ContextLoader
 * @see #ContextLoaderListener(WebApplicationContext)
 * @see #contextInitialized(ServletContextEvent)
 * @see #contextDestroyed(ServletContextEvent)
 */
public ContextLoaderListener() {
}

/**
 * Create a new {@code ContextLoaderListener} with the given application context. This
 * constructor is useful in Servlet 3.0+ environments where instance-based
 * registration of listeners is possible through the {@link javax.servlet.ServletContext#addListener}
 * API. 使用给定的应用上下文创建一个ContextLoaderListener，这个构造方法在Servlet3.0+环境中非常有用，尽可能的通过ServletContext#addListener API 基于实例注册监听器
 * <p>The context may or may not yet be {@linkplain
 * org.springframework.context.ConfigurableApplicationContext#refresh() refreshed}. If it
 * (a) is an implementation of {@link ConfigurableWebApplicationContext} and 
 * (b) has <strong>not</strong> already been refreshed (the recommended approach 推荐方法),
 * then the following will occur: 如果ConfigurableWebApplicationContext的实现还没有刷新时将发生如西行为
 * <ul> 给定的上下文还没有分配一个id，一个将被分配ServletContext和ServletConfig对象将委托给这个application contex,customizeContext()将被调用
 * <li>If the given context has not already been assigned an {@linkplain
 * org.springframework.context.ConfigurableApplicationContext#setId id}, one will be assigned to it</li>
 * <li>{@code ServletContext} and {@code ServletConfig} objects will be delegated to
 * the application context</li>
 * <li>{@link #customizeContext} will be called</li>
 * <li>Any {@link org.springframework.context.ApplicationContextInitializer ApplicationContextInitializer}s
 * specified through the "contextInitializerClasses" init-param will be applied.</li> 任何指定的ApplicationContextInitializer通过contextInitializerClasses初始化参数将被应用,refresh()将被调用
 * <li>{@link org.springframework.context.ConfigurableApplicationContext#refresh refresh()} will be called</li>
 * </ul>
 * If the context has already been refreshed or does not implement 如果上下文已经刷新或者没有ConfigurableWebApplicationContext没有实现，上述情况都不会发生
 * {@code ConfigurableWebApplicationContext}, none of the above will occur under the
 * assumption 假设 that the user has performed these actions (or not) per his or her
 * specific needs.
 * <p>See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
 * <p>In any cas 无论如何e, the given application context will be registered into the 给定的上下文将被注册到ServletContext的属性名上
 * ServletContext under the attribute name {@link
 * WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE} and the Spring
 * application context will be closed when the {@link #contextDestroyed} lifecycle
 * method is invoked on this listener.
 * @param context the application context to manage
 * @see #contextInitialized(ServletContextEvent)
 * @see #contextDestroyed(ServletContextEvent)
 */
public ContextLoaderListener(WebApplicationContext context) {
	super(context);
}
```

- ServletContextListener接口实现

contextInitialized(ServletContextEvent event)执行流程如下：
![](/img/post.img/spring/ContextLoaderListener.contextInitialized.png)

```java
/**
 * Initialize the root web application context. 根Web应用上下文件初始化
 */
@Override
public void contextInitialized(ServletContextEvent event) {
	initWebApplicationContext(event.getServletContext());
}


/**
 * Close the root web application context. 根Web应用上下文件关闭时
 */
@Override
public void contextDestroyed(ServletContextEvent event) {
	closeWebApplicationContext(event.getServletContext());
	ContextCleanupListener.cleanupAttributes(event.getServletContext()); //清楚属性
}

```

## ContextLoader：上下文加载器

ContextLoader为根应用程序上下文执行实际的初始化工作，由ContextLoaderListener调用

- 在web.xml中查找context-param参数级别指定的`contextClass`参数的类型。如果没有找到使用XmlWebApplicationContext【[具体实现过程参考:Spring源码-XmlWebApplicationContext xml应用上下文](/2019/08/01/spring-XmlWebApplicationContext/)】。使用默认的ContextLoader实现，任何指定的上下文类需要实现ConfigurableWebApplicationContext接口

- 处理一个`contextConfigLocation`的context-param参数且将其值传递给上下文实例，将其解析为多个文件路径，可以使用多个`,`和空格分割，如果没有明确的指定，上下文实现支持在使用一个默认的位置(with XmlWebApplicationContext: "/WEB-INF/applicationContext.xml")。在多配置位置场景时，后定义的bean将覆盖以前加载的文件中的定义，至少在使用一个默认的Spring ApplicationContext的实现，通过一个额外的XML文件故意覆盖某些bean定义是有作用的

- 除了加载根应用程序上下文之外，ContextLoader能随意加载或获取共享父上下文并将其连接到根应用程序上下文

- 从Spring 3.1后，ContextLoader支持通过一个ContextLoader(WebApplicationContext)构造方法注册一个根web应用上下文。在Servlet 3.0环境中，允许编程配置


### ContextLoader类说明

```java
/**
 * Performs the actual initialization work for the root application context.
 * Called by {@link ContextLoaderListener}.
 *
 * <p>Looks for a {@link #CONTEXT_CLASS_PARAM "contextClass"} parameter at the
 * {@code web.xml} context-param level to specify the context class type, falling
 * back to {@link org.springframework.web.context.support.XmlWebApplicationContext}
 * if not found. With the default ContextLoader implementation, any context class
 * specified needs to implement the {@link ConfigurableWebApplicationContext} interface.
 *
 * <p>Processes a {@link #CONFIG_LOCATION_PARAM "contextConfigLocation"} context-param
 * and passes its value to the context instance, parsing it into potentially multiple
 * file paths which can be separated by any number of commas and spaces, e.g.
 * "WEB-INF/applicationContext1.xml, WEB-INF/applicationContext2.xml".
 * Ant-style path patterns are supported as well, e.g.
 * "WEB-INF/*Context.xml,WEB-INF/spring*.xml" or "WEB-INF/&#42;&#42;/*Context.xml".
 * If not explicitly specified, the context implementation is supposed to use a
 * default location (with XmlWebApplicationContext: "/WEB-INF/applicationContext.xml").
 *
 * <p>Note: In case of multiple config locations, later bean definitions will
 * override ones defined in previously loaded files, at least when using one of
 * Spring's default ApplicationContext implementations. This can be leveraged
 * to deliberately override certain bean definitions via an extra XML file.
 *
 * <p>Above and beyond loading the root application context, this class can optionally
 * load or obtain and hook up a shared parent context to the root application context.
 * See the {@link #loadParentContext(ServletContext)} method for more information.
 *
 * <p>As of Spring 3.1, {@code ContextLoader} supports injecting the root web
 * application context via the {@link #ContextLoader(WebApplicationContext)}
 * constructor, allowing for programmatic configuration in Servlet 3.0+ environments.
 * See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
 *
 * @author Juergen Hoeller
 * @author Colin Sampaleanu
 * @author Sam Brannen
 * @since 17.02.2003
 * @see ContextLoaderListener
 * @see ConfigurableWebApplicationContext
 * @see org.springframework.web.context.support.XmlWebApplicationContext
 */
public class ContextLoader {

}
```

### ContextLoader成员变量

```java
/**
 * Config param for the root WebApplicationContext id, 根WebApplicationContext id的配置参数
 * to be used as serialization id for the underlying BeanFactory: {@value} 用作底层BeanFactory的序列化id
 */
 //根WebApplicationContext id的配置参数
public static final String CONTEXT_ID_PARAM = "contextId";

/**
 * Name of servlet context parameter (i.e., {@value}) that can specify the 
 * config location for the root context, falling back to the implementation's
 * default otherwise.
 * @see org.springframework.web.context.support.XmlWebApplicationContext#DEFAULT_CONFIG_LOCATION
 */
 //Servlet上下文参数名，用于配置根上下文的指定的Servlet配置文件
public static final String CONFIG_LOCATION_PARAM = "contextConfigLocation";

/**
 * Config param for the root WebApplicationContext implementation class to use: {@value} 
 * @see #determineContextClass(ServletContext)
 */
 //用于根WebApplicationContext实现类的配置参数
public static final String CONTEXT_CLASS_PARAM = "contextClass";

/**
 * Config param for {@link ApplicationContextInitializer} classes to use 
 * for initializing the root web application context: {@value}
 * @see #customizeContext(ServletContext, ConfigurableWebApplicationContext)
 */
 //ApplicationContextInitializer类的配置参数，用于初始化根web应用上下
public static final String CONTEXT_INITIALIZER_CLASSES_PARAM = "contextInitializerClasses";

/**
 * Config param for global {@link ApplicationContextInitializer} classes to use
 * for initializing all web application contexts in the current application: {@value}
 * @see #customizeContext(ServletContext, ConfigurableWebApplicationContext)
 */
 //全局ApplicationContextInitializer类配置参数，用于在当前应用中初始化所有的web应用上下文
public static final String GLOBAL_INITIALIZER_CLASSES_PARAM = "globalInitializerClasses";

/**
 * Any number of these characters are considered delimiters between
 * multiple values in a single init-param String value.
 */
 //初始化参数分隔符
private static final String INIT_PARAM_DELIMITERS = ",; \t\n"; 

/**
 * Name of the class path resource (relative to the ContextLoader class)
 * that defines ContextLoader's default strategy names.
 */
 //类路径资源名(相对于ContextLoader类)，定义ContextLoader的默认策略名
private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";


private static final Properties defaultStrategies;

static {
	// Load default strategy implementations from properties file. 从属性文件加载默认策略实现
	// This is currently strictly internal and not meant to be customized 内部直接使用
	// by application developers.
	try {
		ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
		defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
	}
	catch (IOException ex) {
		throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
	}
}


/**
 * Map from (thread context) ClassLoader to corresponding 'current' WebApplicationContext.
 */
//映射类加载器响应的WebApplicationContext
private static final Map<ClassLoader, WebApplicationContext> currentContextPerThread =
		new ConcurrentHashMap<>(1);

/**
 * The 'current' WebApplicationContext, if the ContextLoader class is 如果ContextLoader类部署在web app类加载器自身 
 * deployed in the web app ClassLoader itself.
 */
@Nullable
private static volatile WebApplicationContext currentContext;  //线程可见


/**
 * The root WebApplicationContext instance that this loader manages. 加载器管理的根WebApplicationContext实例
 */
@Nullable
private WebApplicationContext context;

/** Actual ApplicationContextInitializer instances to apply to the context */ 
//实际ApplicationContextInitializer实例应用在上下文中
private final List<ApplicationContextInitializer<ConfigurableApplicationContext>> contextInitializers =
		new ArrayList<>();
```

ContextLoader.properties文件内容如下：

```java
# Default WebApplicationContext implementation class for ContextLoader.
# Used as fallback when no explicit context implementation has been specified as context-param.
# Not meant to be customized by application developers. 不打算由应用程序开发人员定制

org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext

```


### ContextLoader构造方法

```java
/**
 * Create a new {@code ContextLoader} that will create a web application context
 * based on the "contextClass" and "contextConfigLocation" servlet context-params.
 * See class-level documentation for details on default values for each.
 * <p>This constructor is typically used when declaring the {@code
 * ContextLoaderListener} subclass as a {@code <listener>} within {@code web.xml}, as
 * a no-arg constructor is required.
 * <p>The created application context will be registered into the ServletContext under
 * the attribute name {@link WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE}
 * and subclasses are free to call the {@link #closeWebApplicationContext} method on
 * container shutdown to close the application context.
 * @see #ContextLoader(WebApplicationContext)
 * @see #initWebApplicationContext(ServletContext)
 * @see #closeWebApplicationContext(ServletContext)
 */
public ContextLoader() {
}

/**
 * Create a new {@code ContextLoader} with the given application context. This
 * constructor is useful in Servlet 3.0+ environments where instance-based
 * registration of listeners is possible through the {@link ServletContext#addListener}
 * API.
 * <p>The context may or may not yet be {@linkplain
 * ConfigurableApplicationContext#refresh() refreshed}. If it (a) is an implementation
 * of {@link ConfigurableWebApplicationContext} and (b) has <strong>not</strong>
 * already been refreshed (the recommended approach), then the following will occur:
 * <ul>
 * <li>If the given context has not already been assigned an {@linkplain
 * ConfigurableApplicationContext#setId id}, one will be assigned to it</li>
 * <li>{@code ServletContext} and {@code ServletConfig} objects will be delegated to
 * the application context</li>
 * <li>{@link #customizeContext} will be called</li>
 * <li>Any {@link ApplicationContextInitializer}s specified through the
 * "contextInitializerClasses" init-param will be applied.</li>
 * <li>{@link ConfigurableApplicationContext#refresh refresh()} will be called</li>
 * </ul>
 * If the context has already been refreshed or does not implement
 * {@code ConfigurableWebApplicationContext}, none of the above will occur under the
 * assumption that the user has performed these actions (or not) per his or her
 * specific needs.
 * <p>See {@link org.springframework.web.WebApplicationInitializer} for usage examples.
 * <p>In any case, the given application context will be registered into the
 * ServletContext under the attribute name {@link
 * WebApplicationContext#ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE} and subclasses are
 * free to call the {@link #closeWebApplicationContext} method on container shutdown
 * to close the application context.
 * @param context the application context to manage
 * @see #initWebApplicationContext(ServletContext)
 * @see #closeWebApplicationContext(ServletContext)
 */
public ContextLoader(WebApplicationContext context) {
	this.context = context;
}

```

### ContextLoader核心方法

- setContextInitializers(@Nullable ApplicationContextInitializer<?>... initializers)

将ApplicationContextInitializer接口的实现类加入到contextInitializers列表中，在应用上下文中通过ContextLoader初始化这些实例

```java
/**
 * Specify which {@link ApplicationContextInitializer} instances should be used
 * to initialize the application context used by this {@code ContextLoader}.
 * @since 4.2
 * @see #configureAndRefreshWebApplicationContext
 * @see #customizeContext
 */
@SuppressWarnings("unchecked")
public void setContextInitializers(@Nullable ApplicationContextInitializer<?>... initializers) {
	if (initializers != null) {
		for (ApplicationContextInitializer<?> initializer : initializers) {
			this.contextInitializers.add((ApplicationContextInitializer<ConfigurableApplicationContext>) initializer);
		}
	}
}
```


- initWebApplicationContext(ServletContext servletContext)

从给定的ServletContext中初始化Spring的web应用上下文，使用构建时提供的应用上下文或者根据contextClass和contextConfigLocation context-params参数创建一个新的应用上下文


```java
	/**
	 * Initialize Spring's web application context for the given servlet context,
	 * using the application context provided at construction time, or creating a new one
	 * according to the "{@link #CONTEXT_CLASS_PARAM contextClass}" and
	 * "{@link #CONFIG_LOCATION_PARAM contextConfigLocation}" context-params.
	 * @param servletContext current servlet context 当前servletcontext
	 * @return the new WebApplicationContext
	 * @see #ContextLoader(WebApplicationContext)
	 * @see #CONTEXT_CLASS_PARAM
	 * @see #CONFIG_LOCATION_PARAM
	 */
	public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		//检查ServletContext是否包含WebApplicationContext属性，存在时则抛非法状态异常
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		Log logger = LogFactory.getLog(ContextLoader.class);
		servletContext.log("Initializing Spring root WebApplicationContext");
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that 将上下文存储在本地实例变量中，确保
			// it is available on ServletContext shutdown. 在ServletContext关闭时可用
			if (this.context == null) {  //WebApplicationContext为null时创建
				this.context = createWebApplicationContext(servletContext);
			}
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) { //配置web应用上下文没有激活，即还没有有调用refreshed非法或者关闭
					// The context has not yet been refreshed -> provide services such as 提供服务如设置父上下文，设置应用上下文id
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) { //父上下文为null
						//下文实例在没有显式paren的情况下被注入
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any. 确定根web应用程序上下文的父类
						ApplicationContext parent = loadParentContext(servletContext); //模板方法加载父上下，默认返回null，由子类重写
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext); //配置且刷新web应用上下文
				}
			}
			//ServletContext上下文中设置WebApplicationContext实例
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader(); //当前线程类加载器
			if (ccl == ContextLoader.class.getClassLoader()) { //相同时设置currentContext为当前的WebApplicationContext
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context); //映射类加载器为当前的WebApplicationContext
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
						WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
			}
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

			return this.context; //返回
		}
		catch (RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
		catch (Error err) {
			logger.error("Context initialization failed", err);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
			throw err;
		}
	}

```

- createWebApplicationContext(ServletContext sc)

由默认的上下文类或者定制的上下文类来创建根WebApplicationContext对象

```java
	/**
	 * Instantiate the root WebApplicationContext for this loader, either the
	 * default context class or a custom context class if specified.
	 * <p>This implementation expects custom contexts to implement the 实现这期望定制的contexts实现ConfigurableWebApplicationContext接口
	 * {@link ConfigurableWebApplicationContext} interface.
	 * Can be overridden in subclasses.
	 * <p>In addition, 另外{@link #customizeContext} gets called prior to refreshing the customizeContext()在刷新上下文时被调用
	 * context, allowing subclasses to perform custom modifications to the context.允许子类对上下文执行自定义修改
	 * @param sc current servlet context
	 * @return the root WebApplicationContext
	 * @see ConfigurableWebApplicationContext
	 */
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		Class<?> contextClass = determineContextClass(sc); //确定上下文类
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) { //是否继承自ConfigurableWebApplicationContext
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass); //实例化ConfigurableWebApplicationContext接口的实现
	}
```

- determineContextClass(ServletContext servletContext)

返回WebApplicationContext接口的实现类，使用默认的XmlWebApplicationContext或者定制的context class 

```java
	/**
	 * Return the WebApplicationContext implementation class to use, either the
	 * default XmlWebApplicationContext or a custom context class if specified.
	 * @param servletContext current servlet context
	 * @return the WebApplicationContext implementation class to use
	 * @see #CONTEXT_CLASS_PARAM
	 * @see org.springframework.web.context.support.XmlWebApplicationContext
	 */
	protected Class<?> determineContextClass(ServletContext servletContext) {
		String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM); //ServletContext属性中获取contextClass名
		if (contextClassName != null) { //非空时使用默认的类加载器加载
			try {
				return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader()); 
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load custom context class [" + contextClassName + "]", ex);
			}
		}
		else {
			//从配置属性中获取默认的上下文实现类：XmlWebApplicationContext
			contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
			try {
				return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader()); //加载
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load default context class [" + contextClassName + "]", ex);
			}
		}
	}
```

- configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc)

配置且刷新WebApplicationContext，`先调用ConfigurableWebEnvironment#initPropertySources(sc, null)初始化Servlet属性源，然后再调用定制上下文customizeContext方法，最后执行ConfigurableWebApplicationContext#refresh方法`

```java
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value 应用程序上下文id仍然设置为其原始默认值
			// -> assign a more useful id based on available information 根据可用信息分配更有用的id
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM); //ServletContext属性读取contextId
			if (idParam != null) {
				wac.setId(idParam);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath())); //WebApplicationContext.class.getName() + ":"+上下文路径
			}
		}

		wac.setServletContext(sc); //ConfigurableWebApplicationContext设置ServletContext
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM); //获取Spring配置路径参数
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);//ConfigurableWebApplicationContext设置配置参数
		}

		// The wac environment's #initPropertySources will be called in any case when the context 任何情况下，在上下文刷新时，environment的initPropertySources初始化属性源方法将被调用
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for 这里连接执行initPropertySources方法，在任何后置处理器或者在#refresh之前初始化，以确保servlet属性源准备使用
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		customizeContext(sc, wac);
		wac.refresh();
	}

```

- customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac)

定制ConfigurableWebApplicationContext的创建，在配置提供的locations之后，context刷新之前调用。默认实现为从ServletContext属性参数`CONTEXT_INITIALIZER_CLASSES_PARAM`中获取所有的实现接口的类，使用Order接口或@Order注解排序后依次执行void initialize(C extends ConfigurableApplicationContext applicationContext)方法。ApplicationContextInitializer接口详细说明见:[Spring源码-ApplicationContextInitializer应用上下文初始化接口](/2019/08/04/spring-ApplicationContextInitializer/)



```java
	/**
	 * Customize the {@link ConfigurableWebApplicationContext} created by this
	 * ContextLoader after config locations have been supplied to the context
	 * but before the context is <em>refreshed</em>.
	 * <p>The default implementation {@linkplain #determineContextInitializerClasses(ServletContext)
	 * determines} what (if any) context initializer classes have been specified through
	 * {@linkplain #CONTEXT_INITIALIZER_CLASSES_PARAM context init parameters} and
	 * {@linkplain ApplicationContextInitializer#initialize invokes each} with the
	 * given web application context.
	 * <p>Any {@code ApplicationContextInitializers} implementing
	 * {@link org.springframework.core.Ordered Ordered} or marked with @{@link
	 * org.springframework.core.annotation.Order Order} will be sorted appropriately 适当地.
	 * @param sc the current servlet context
	 * @param wac the newly created application context
	 * @see #CONTEXT_INITIALIZER_CLASSES_PARAM
	 * @see ApplicationContextInitializer#initialize(ConfigurableApplicationContext)
	 */
	protected void customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac) {
		//确认上下文初始化类列表
		List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> initializerClasses =
				determineContextInitializerClasses(sc);

		for (Class<ApplicationContextInitializer<ConfigurableApplicationContext>> initializerClass : initializerClasses) {
			Class<?> initializerContextClass =
					GenericTypeResolver.resolveTypeArgument(initializerClass, ApplicationContextInitializer.class); //解析ApplicationContextInitializer的泛型是否为ConfigurableApplicationContext
			if (initializerContextClass != null && !initializerContextClass.isInstance(wac)) {
				throw new ApplicationContextException(String.format(
						"Could not apply context initializer [%s] since its generic parameter [%s] " +
						"is not assignable from the type of application context used by this " +
						"context loader: [%s]", initializerClass.getName(), initializerContextClass.getName(),
						wac.getClass().getName()));
			}
			this.contextInitializers.add(BeanUtils.instantiateClass(initializerClass));//实例化ApplicationContextInitializer<C extends ConfigurableApplicationContext>接口实现类
		}

		AnnotationAwareOrderComparator.sort(this.contextInitializers); //排序
		for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
			initializer.initialize(wac); //初始化
		}
	}


```

- determineContextInitializerClasses(ServletContext servletContext)

确认上下初始化类

```java
	/**
	 * Return the {@link ApplicationContextInitializer} implementation classes to use 使用指定的contextInitializerClasses参数，返回所有的ApplicationContextInitializer接口实现类
	 * if any have been specified by {@link #CONTEXT_INITIALIZER_CLASSES_PARAM}.
	 * @param servletContext current servlet context
	 * @see #CONTEXT_INITIALIZER_CLASSES_PARAM
	 */
	protected List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>
			determineContextInitializerClasses(ServletContext servletContext) {

		List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> classes =
				new ArrayList<>();

		String globalClassNames = servletContext.getInitParameter(GLOBAL_INITIALIZER_CLASSES_PARAM);//ServletContext中获取globalInitializerClasses参数指定的全局类名，使用,或者;或者\t 制表符或者\n换行符分割

		if (globalClassNames != null) {
			for (String className : StringUtils.tokenizeToStringArray(globalClassNames, INIT_PARAM_DELIMITERS)) {//换行符分割类名
				classes.add(loadInitializerClass(className));
			}
		}

		String localClassNames = servletContext.getInitParameter(CONTEXT_INITIALIZER_CLASSES_PARAM); //contextInitializerClasses参数指定的类名
		if (localClassNames != null) {
			for (String className : StringUtils.tokenizeToStringArray(localClassNames, INIT_PARAM_DELIMITERS)) {
				classes.add(loadInitializerClass(className));
			}
		}

		return classes;
	}

	//加载上下文初始化类并进行类型检查(是否为ApplicationContextInitializer子类)
	@SuppressWarnings("unchecked") 
	private Class<ApplicationContextInitializer<ConfigurableApplicationContext>> loadInitializerClass(String className) {
		try {
			Class<?> clazz = ClassUtils.forName(className, ClassUtils.getDefaultClassLoader());
			if (!ApplicationContextInitializer.class.isAssignableFrom(clazz)) {
				throw new ApplicationContextException(
						"Initializer class does not implement ApplicationContextInitializer interface: " + clazz);
			}
			return (Class<ApplicationContextInitializer<ConfigurableApplicationContext>>) clazz;
		}
		catch (ClassNotFoundException ex) {
			throw new ApplicationContextException("Failed to load context initializer class [" + className + "]", ex);
		}
	}
```

- loadParentContext(ServletContext servletContext)

模板方法加载父上下文，Spring 5.x默认返回null

```java
	/**
	 * Template method with default implementation (which may be overridden by a
	 * subclass), to load or obtain an ApplicationContext instance which will be 加载或者获取ApplicationContext实例，用于父根
	 * used as the parent context of the root WebApplicationContext. If the  WebApplicationContext的父上下。如何返回null，没有父上下
	 * return value from the method is null, no parent context is set.  文设置
	 * <p>The main reason to load a parent context here is to allow multiple root 主要的原因是加载父上下文允许多个根
	 * web application contexts to all be children of a shared EAR context, or   所有web应用程序上下文都是共享EAR上下文的子上下文
	 * alternately to also share the same parent context that is visible to
	 * EJBs. For pure web applications, there is usually no need to worry about 对于纯web应用程序，通常没有必要担心有一个父上下在根WebApplicationContext
	 * having a parent context to the root web application context.
	 * <p>The default implementation simply returns {@code null}, as of 5.0. 
	 * @param servletContext current servlet context
	 * @return the parent application context, or {@code null} if none
	 */
	@Nullable
	protected ApplicationContext loadParentContext(ServletContext servletContext) {
		return null;
	}
```


- closeWebApplicationContext(ServletContext servletContext)

从servletContext关闭WebApplicationContext，如果从写了loadParentContext(ServletContext)方法，可能也需要重写当前方法

```java
	/**
	 * Close Spring's web application context for the given servlet context.
	 * <p>If overriding {@link #loadParentContext(ServletContext)}, you may have
	 * to override this method as well.
	 * @param servletContext the ServletContext that the WebApplicationContext runs in
	 */
	public void closeWebApplicationContext(ServletContext servletContext) {
		servletContext.log("Closing Spring root WebApplicationContext");
		try {
			if (this.context instanceof ConfigurableWebApplicationContext) {
				((ConfigurableWebApplicationContext) this.context).close(); //调用ConfigurableWebApplicationContext的close方法
			}
		}
		finally {
			ClassLoader ccl = Thread.currentThread().getContextClassLoader(); 
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = null;
			}
			else if (ccl != null) {
				currentContextPerThread.remove(ccl); //移除当前映射中的上下类文加载器
			}
			servletContext.removeAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);//删除ServletContext中WebApplicationContext属性值
		}
	}

```

- getCurrentWebApplicationContext

从当前线程中获取Spring根的WebApplicationContext

```java
	/**
	 * Obtain the Spring root web application context for the current thread
	 * (i.e. for the current thread's context ClassLoader, which needs to be
	 * the web application's ClassLoader).
	 * @return the current root web application context, or {@code null}
	 * if none found
	 * @see org.springframework.web.context.support.SpringBeanAutowiringSupport
	 */
	@Nullable
	public static WebApplicationContext getCurrentWebApplicationContext() {
		ClassLoader ccl = Thread.currentThread().getContextClassLoader(); //当前线程类加载器
		if (ccl != null) { //不为null时，从映射类加载器的当前上下文件线程中获取，
			WebApplicationContext ccpt = currentContextPerThread.get(ccl);
			if (ccpt != null) {
				return ccpt;
			}
		}
		return currentContext;//返回当前上下文
	}

```
