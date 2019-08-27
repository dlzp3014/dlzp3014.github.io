---
layout: post
title:  "Spring源码-XmlWebApplicationContext web.xml应用上下文实现"
date:   2019-08-01 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Web
---

* content
{:toc}

XmlWebApplicationContext为WebApplicationContext接口的实现，由XmlBeanDefinitionReader[xmlBean定义读取器]从XML文档中获取其配置，本质上等价于GenericXmlApplicationContext[通用xml应用上下文]，只是其应用于web环境中。其类结构如下：

![](\img\post.img\spring\XmlWebApplicationContext.png)


- 默认情况下，配置从`/WEB-INF/applicationContext.xml`获取，用于root context[根上下文]

- 默认的配置路径可以通过org.springframework.web.context.ContextLoader的context-param[上下文参数] 和org.springframework.web.servlet.FrameworkServlet的servlet init-param `contextConfigLocation`覆盖

- 配置路径可以是具体的文件如"/WEB-INF/context.xml"或者为`Ant-style模式`如"/WEB-INF/\*-context.xml"

【Note:】在多配置文件场景下，最后的bean定义将覆盖早些时候加载的一个bean定义。`其作用在于可以利用覆盖故意通过一个额外的XML文件覆盖某些bean定义`








```java
/**
 * {@link org.springframework.web.context.WebApplicationContext} implementation
 * which takes its configuration from XML documents, understood by an
 * {@link org.springframework.beans.factory.xml.XmlBeanDefinitionReader}.
 * This is essentially 本质上 the equivalent of
 * {@link org.springframework.context.support.GenericXmlApplicationContext}
 * for a web environment.
 *
 * <p>By default, the configuration will be taken from "/WEB-INF/applicationContext.xml"
 * for the root context, and "/WEB-INF/test-servlet.xml" for a context with the namespace
 * "test-servlet" (like for a DispatcherServlet instance with the servlet-name "test").
 *
 * <p>The config location defaults can be overridden via the "contextConfigLocation"
 * context-param of {@link org.springframework.web.context.ContextLoader} and servlet
 * init-param of {@link org.springframework.web.servlet.FrameworkServlet}. Config locations
 * can either denote concrete files like "/WEB-INF/context.xml" or Ant-style patterns
 * like "/WEB-INF/*-context.xml" (see {@link org.springframework.util.PathMatcher}
 * javadoc for pattern details).
 *
 * <p>Note: In case of multiple config locations, later bean definitions will
 * override ones defined in earlier loaded files. This can be leveraged to
 * deliberately override certain bean definitions via an extra XML file.
 *
 * <p><b>For a WebApplicationContext that reads in a different bean definition format, 对于以不同bean定义格式读取的WebApplicationContext,创建一个类似的子类
 * create an analogous subclass of {@link AbstractRefreshableWebApplicationContext}.</b>
 * Such a context implementation can be specified as "contextClass" context-param 上下文实现可以指定一个contextClass实现类用于ContextLoader或者FrameworkServlet中
 * for ContextLoader or "contextClass" init-param for FrameworkServlet.
 *
 * @see #setNamespace
 * @see #setConfigLocations
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see org.springframework.web.context.ContextLoader#initWebApplicationContext
 * @see org.springframework.web.servlet.FrameworkServlet#initWebApplicationContext
 */
public class XmlWebApplicationContext extends AbstractRefreshableWebApplicationContext {

	/** Default config location for the root context */ 默认配置路径
	public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";

	/** Default prefix for building a config location for a namespace */ 默认构建配置路径的命名空间
	public static final String DEFAULT_CONFIG_LOCATION_PREFIX = "/WEB-INF/";

	/** Default suffix for building a config location for a namespace */ 后缀
	public static final String DEFAULT_CONFIG_LOCATION_SUFFIX = ".xml";

	通过XmlBeanDefinitionReader加载bean定义
	/**
	 * Loads the bean definitions via an XmlBeanDefinitionReader. 
	 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
	 * @see #initBeanDefinitionReader
	 * @see #loadBeanDefinitions
	 */
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory. 
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(getEnvironment());  环境
		beanDefinitionReader.setResourceLoader(this); 资源加载器
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this)); 资源实体解析器

		// Allow a subclass to provide custom initialization of the reader, 允许子类提供自定义初始化读取器
		// then proceed with actually loading the bean definitions. 然后继续实际加载bean定义
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

	初始化一个bean definition 读取器，用于加载bean definitions 到上下文中，默认实现为空
	/**
	 * Initialize the bean definition reader used for loading the bean 
	 * definitions of this context. Default implementation is empty.
	 * <p>Can be overridden in subclasses 可以被子类重写, e.g. for turning off XML validation 关闭xml校验
	 * or using a different XmlBeanDefinitionParser implementation. 或者使用不同的XmlBeanDefinitionParser解析器
	 * @param beanDefinitionReader the bean definition reader used by this context
	 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader#setValidationMode
	 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader#setDocumentReaderClass
	 */
	protected void initBeanDefinitionReader(XmlBeanDefinitionReader beanDefinitionReader) {
	}

	使用给定的XmlBeanDefinitionReader加载 bean definitions
	/**
	 * Load the bean definitions with the given XmlBeanDefinitionReader. bean工厂的声明周期由refreshBeanFactory方法控制
	 * <p>The lifecycle of the bean factory  is handled by the refreshBeanFactory method; 
	 * therefore this method is just supposed to load and/or register bean definitions. 因此当前方法只需要加载或者注册一个bean定义 ，委托给ResourcePatternResolver资源模式解析器去解析位置模式为一个资源实例
	 * <p>Delegates to a ResourcePatternResolver for resolving location patterns
	 * into Resource instances.
	 * @throws IOException if the required XML document isn't found
	 * @see #refreshBeanFactory
	 * @see #getConfigLocations
	 * @see #getResources
	 * @see #getResourcePatternResolver
	 */
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations(); //多配置路径
		if (configLocations != null) {
			for (String configLocation : configLocations) { 循环读取
				reader.loadBeanDefinitions(configLocation); 
			}
		}
	}

	默认配置路径
	/**
	 * The default location for the root context is "/WEB-INF/applicationContext.xml",
	 * and "/WEB-INF/test-servlet.xml" for a context with the namespace "test-servlet"
	 * (like for a DispatcherServlet instance with the servlet-name "test").
	 */
	@Override
	protected String[] getDefaultConfigLocations() {
		if (getNamespace() != null) {
			return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
		}
		else {
			return new String[] {DEFAULT_CONFIG_LOCATION};
		}
	}

}

```
