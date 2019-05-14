---
layout: post
title:  "Spring源码-ResourceLoader资源加载器"
date:   2019-04-07 00:46:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

Spring中ResourceLoader接口为策略接口，用于加载各种资源文件，如类路径资源或者文件系统资源

ResourcePatternResolver接口扩展了ResourceLoader，用于获取可匹配模式的多个资源。

类结构图如下：

![](/img/post.img/spring/ResourceLoader-uml.png)



## ResourceLoader接口定义

```java

/**
 * Strategy interface for loading resources (e.. class path or file system
 * resources). An {@link org.springframework.context.ApplicationContext}  ApplicationContext需要提供这个功能，扩展
 * is required to provide this functionality, plus extended  ResourcePatternResolver的支持
 * {@link org.springframework.core.io.support.ResourcePatternResolver} support.
 *
 * <p>{@link DefaultResourceLoader} is a standalone implementation that is DefaultResourceLoader是一个单独实现，
 * usable outside an ApplicationContext, also used by {@link ResourceEditor}. 能够在ApplicationContext外使用，也被
 *																		用于ResourceEditor
 * <p>Bean properties of type Resource and Resource array can be populated  当运行在ApplicationContext中时，资源类型的Bean属性和数组资源从字符串填充
 * from Strings when running in an ApplicationContext, using the particular 使用特定的上下文资源加载策略
 * context's resource loading strategy.
 *
 * @author Juergen Hoeller
 * @since 10.03.2004
 * @see Resource
 * @see org.springframework.core.io.support.ResourcePatternResolver 资源模式解析器
 * @see org.springframework.context.ApplicationContext 应用上下文
 * @see org.springframework.context.ResourceLoaderAware 
 */
public interface ResourceLoader {

	/** Pseudo URL prefix for loading from the class path: "classpath:" */ 伪URL前缀用于从类路径中加载资源
	String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;


	/**
	 * Return a Resource handle for the specified resource location. 返回指定资源位置的资源句柄
	 * <p>The handle should always be a reusable resource descriptor, 资源句柄始终是可重用的资源描述符，运行多次调用
	 * allowing for multiple {@link Resource#getInputStream()} calls.
	 * <p><ul>
	 * <li>Must support fully qualified URLs, e.g. "file:C:/test.dat".  支持完全限定的url
	 * <li>Must support classpath pseudo-URLs, e.g. "classpath:test.dat". 支持类路径
	 * <li>Should support relative file paths, e.g. "WEB-INF/test.dat". 支持相对稳健路径
	 * (This will be implementation-specific, typically provided by an 特定实现，通常由ApplicationContext实现提供
	 * ApplicationContext implementation.)
	 * </ul>
	 * <p>Note that a Resource handle does not imply an existing resource; 资源句柄并不意味着一个存在的资源
	 * you need to invoke {@link Resource#exists} to check for existence. 需要调用exists检查存在
	 * @param location the resource location 资源位置
	 * @return a corresponding Resource handle (never {@code null}) 返回一个响应的资源句柄，否则返回null
	 * @see #CLASSPATH_URL_PREFIX
	 * @see Resource#exists()
	 * @see Resource#getInputStream()
	 */
	Resource getResource(String location);

	/**
	 * Expose the ClassLoader used by this ResourceLoader. 公开这个ResourceLoader使用的类加载器
	 * <p>Clients which need to access the ClassLoader directly can do so 需要直接访问类加载器的客户端，可以
	 * in a uniform manner with the ResourceLoader, rather than relying 以统一的方式与ResourceLoader进行此操作
	 * on the thread context ClassLoader. 而不是依赖线程上下文类加载器
	 * @return the ClassLoader 
	 * (only {@code null} if even the system ClassLoader isn't accessible) 如果连系统类加载器都不可访问返回null
	 * @see org.springframework.util.ClassUtils#getDefaultClassLoader() 使用默认的类加载器
	 * @see org.springframework.util.ClassUtils#forName(String, ClassLoader)
	 */
	@Nullable
	ClassLoader getClassLoader();

}

```

## Spring提供的实现

### DefaultResourceLoader:资源加载器

ResourceLoader默认实现，通过类加载器加载资源，内部定义了私有的静态类ClassPathContextResource(类路径上下文件资源)，可使用指定的ClassLoader来加载资源，且使用ConcurrentHashMap缓存已加载过的资源。

Spring 4.3后可以使用ProtocolResolver协议解析器来解析资源，ProtocolResolver是一个函数式接口，定义如下：

```java
/**
 * A resolution strategy for protocol-specific resource handles. 用于指定协议资源处理的解决策略
 *
 * <p>Used as an SPI for {@link DefaultResourceLoader}, allowing for 用作DefaultResourceLoader的SPI(服务提供接口)，允许自定义
 * custom protocols to be handled without subclassing the loader 协议处理没有子类的加载器实现(应用程序上下文实现)
 * implementation (or application context implementation).
 *
 * @author Juergen Hoeller
 * @since 4.3
 * @see DefaultResourceLoader#addProtocolResolver
 */
@FunctionalInterface
public interface ProtocolResolver {

	/**
	 * Resolve the given location against the given resource loader 根据给定的资源加载器解析给定的位置
	 * if this implementation's protocol matches. 如果此实现的协议匹配
	 * @param location the user-specified resource location 指定的资源位置
	 * @param resourceLoader the associated resource loader 关联的资源加载器
	 * @return a corresponding {@code Resource} handle if the given location
	 * matches this resolver's protocol, or {@code null} otherwise
	 */
	@Nullable
	Resource resolve(String location, ResourceLoader resourceLoader);

}
```

DefaultResourceLoader中Resource getResource(String location)方法具体使用过程为：

- 使用注册进来的ProtocolResolver去解析资源(resolve())，如果不为null，直接返回

- location以"/"开头，使用ClassPathContextResource(类路径上下文资源)加载资源。ClassPathContextResource继承了ClassPathResource实现了ContextResource(上下文)接口

- location以"classpath:"开头时，使用ClassPathResource类路径资源加载器

- 其他情况时将location构建为URL对象，URL对象构建成功时，如果URL为一个文件url时使用FileUrlResource，否则使用UrlResource加载资源；URL对象构建失败时，使用ClassPathContextResource

```java

/**
 * Default implementation of the {@link ResourceLoader} interface. 
 * Used by {@link ResourceEditor}, and serves as base class for 使用在ResourceEditor和服务于基类AbstractApplicationContext
 * {@link org.springframework.context.support.AbstractApplicationContext}.
 * Can also be used standalone. 可单独使用
 *
 * <p>Will return a {@link UrlResource} if the location value is a URL, 如果位置的值是一个URL，将返回一个UrlResource
 * and a {@link ClassPathResource} if it is a non-URL path or a 如果不是一个URL路径或classpath:伪协议，返回一个
 * "classpath:" pseudo-URL.  ClassPathResource
 *
 * @author Juergen Hoeller
 * @since 10.03.2004
 * @see FileSystemResourceLoader 文件系统资源加载器
 * @see org.springframework.context.support.ClassPathXmlApplicationContext
 */
public class DefaultResourceLoader implements ResourceLoader {

	@Nullable
	private ClassLoader classLoader; //类加载器

	private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4); //可使用的协议解析器

	private final Map<Class<?>, Map<Resource, ?>> resourceCaches = new ConcurrentHashMap<>(4); //缓存已加载的资源


	/**
	 * Create a new DefaultResourceLoader. 创建一个默认的DefaultResourceLoader
	 * <p>ClassLoader access will happen using the thread context class loader 类架起访问将使用线程上下文类加载器
	 * at the time of this ResourceLoader's initialization. 在ResourceLoader初始化时
	 * @see java.lang.Thread#getContextClassLoader()
	 */
	public DefaultResourceLoader() {
		this.classLoader = ClassUtils.getDefaultClassLoader(); //使用默认的类加载器
	}

	/**
	 * Create a new DefaultResourceLoader.
	 * @param classLoader the ClassLoader to load class path resources with, or {@code null} 类加载器用来加载类路径的资源
	 * for using the thread context class loader at the time of actual resource access  在实际访问资源时
	 */
	public DefaultResourceLoader(@Nullable ClassLoader classLoader) {
		this.classLoader = classLoader;
	}


	/**
	 * Specify the ClassLoader to load class path resources with, or {@code null} 指定要加载类路径资源的类加载器
	 * for using the thread context class loader at the time of actual resource access.
	 * <p>The default is that ClassLoader access will happen using the thread context
	 * class loader at the time of this ResourceLoader's initialization.
	 */
	public void setClassLoader(@Nullable ClassLoader classLoader) {
		this.classLoader = classLoader;
	}

	/**
	 * Return the ClassLoader to load class path resources with. 返回类加载器以加载类路径资源
	 * <p>Will get passed to ClassPathResource's constructor for all
	 * ClassPathResource objects created by this resource loader.
	 * @see ClassPathResource
	 */
	@Override
	@Nullable
	public ClassLoader getClassLoader() {
		return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
	}

	/**
	 * Register the given resolver with this resource loader, allowing for 通过资源加载器注册给定的解析器
	 * additional protocols to be handled. 允许附加协议处理
	 * <p>Any such resolver will be invoked ahead of this loader's standard 类似这样的解析器将在这个标准的累加器之前执行
	 * resolution rules. It may therefore also override any default rules. 因此，它也可能覆盖任何默认规则
	 * @since 4.3
	 * @see #getProtocolResolvers()
	 */
	public void addProtocolResolver(ProtocolResolver resolver) {
		Assert.notNull(resolver, "ProtocolResolver must not be null");
		this.protocolResolvers.add(resolver);
	}

	/**
	 * Return the collection of currently registered protocol resolvers, 返回当前注册的协议解析器的集合
	 * allowing for introspection as well as modification. 允许自省和修改
	 * @since 4.3
	 */
	public Collection<ProtocolResolver> getProtocolResolvers() {
		return this.protocolResolvers;
	}

	/**
	 * Obtain a cache for the given value type, keyed by {@link Resource}. 获取给定值类型的缓存，key为Resource
	 * @param valueType the value type, e.g. an ASM {@code MetadataReader} 类型值可以为 MetadataReader
	 * @return the cache {@link Map}, shared at the {@code ResourceLoader} level 返回缓存的Map，共享资源加载器的等级
	 * @since 5.0
	 */
	@SuppressWarnings("unchecked")
	public <T> Map<Resource, T> getResourceCache(Class<T> valueType) {
		return (Map<Resource, T>) this.resourceCaches.computeIfAbsent(valueType, key -> new ConcurrentHashMap<>());
	}

	/**
	 * Clear all resource caches in this resource loader. 清除此资源加载器中的所有资源缓存
	 * @since 5.0
	 * @see #getResourceCache
	 */
	public void clearResourceCaches() {
		this.resourceCaches.clear();
	}


	@Override
	public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");

		for (ProtocolResolver protocolResolver : this.protocolResolvers) { //直接通过协议解析器解析资源，如果解析成功，直接返回
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}

		if (location.startsWith("/")) { //路径以"/"开始时，使用ClassPathContextResource 类路径上下文加载资源
			return getResourceByPath(location); 
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) { //路径以"classpath:"时，使用ClassPathResource加载资源
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				// Try to parse the location as a URL... //通过位置构建URL，如果URL是文件使用FileUrlResource，否则使用UrlResource
				URL url = new URL(location);
				return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
			}
			catch (MalformedURLException ex) {
				// No URL -> resolve as resource path. 无法解析URL时，使用ClassPathContextResource加载资源
				return getResourceByPath(location);
			}
		}
	}

	/**
	 * Return a Resource handle for the resource at the given path. 再给定路径上返回资源处理器的资源
	 * <p>The default implementation supports class path locations. This should 默认实现支持类路径位置，
	 * be appropriate for standalone implementations but can be overridden, 这适用于单独实现，可以重新
	 * e.g. for implementations targeted at a Servlet container. 用于针对Servlet容器的实现
	 * @param path the path to the resource 指向资源的路径
	 * @return the corresponding Resource handle 对应的资源句柄
	 * @see ClassPathResource
	 * @see org.springframework.context.support.FileSystemXmlApplicationContext#getResourceByPath
	 * @see org.springframework.web.context.support.XmlWebApplicationContext#getResourceByPath
	 */
	protected Resource getResourceByPath(String path) {
		return new ClassPathContextResource(path, getClassLoader());
	}


	/**
	 * ClassPathResource that explicitly expresses(明确地表示) a context-relative path 上下文向路径
	 * through implementing the ContextResource interface. 通过实现ContextResource接口，
	 */
	protected static class ClassPathContextResource extends ClassPathResource implements ContextResource {

		public ClassPathContextResource(String path, @Nullable ClassLoader classLoader) {
			super(path, classLoader);
		}

		@Override
		public String getPathWithinContext() {
			return getPath();
		}

		@Override
		public Resource createRelative(String relativePath) {
			String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
			return new ClassPathContextResource(pathToUse, getClassLoader());
		}
	}

}

```

### ClassRelativeResourceLoader：类相对资源加载器

ClassRelativeResourceLoader通过指定的类的类加载器加载资源，其继承DefaultResourceLoader，添加了使用类相对的上下文来加载资源，内部定义了ClassRelativeContextResource(类相对上下文资源)静态内部类


```java
ResourceLoader实现，解释普通资源路径作为给定类的相对路径
/**
 * {@link ResourceLoader} implementation that interprets plain resource paths 
 * as relative to a given {@code java.lang.Class}.
 *
 * @author Juergen Hoeller
 * @since 3.0
 * @see Class#getResource(String)
 * @see ClassPathResource#ClassPathResource(String, Class)
 */
public class ClassRelativeResourceLoader extends DefaultResourceLoader {

	//路径相对应的类
	private final Class<?> clazz;


	/**
	 * Create a new ClassRelativeResourceLoader for the given class. 使用类创建ClassRelativeResourceLoader
	 * @param clazz the class to load resources through 通过类来加载资源
	 */
	public ClassRelativeResourceLoader(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		this.clazz = clazz;
		setClassLoader(clazz.getClassLoader());
	}

	@Override
	protected Resource getResourceByPath(String path) {
		return new ClassRelativeContextResource(path, this.clazz);
	}


	/**
	 * ClassPathResource that explicitly expresses a context-relative path
	 * through implementing the ContextResource interface.
	 */
	private static class ClassRelativeContextResource extends ClassPathResource implements ContextResource {

		private final Class<?> clazz;

		public ClassRelativeContextResource(String path, Class<?> clazz) {
			super(path, clazz);
			this.clazz = clazz;
		}

		@Override
		public String getPathWithinContext() {
			return getPath();
		}

		@Override
		public Resource createRelative(String relativePath) {
			String pathToUse = StringUtils.applyRelativePath(getPath(), relativePath);
			return new ClassRelativeContextResource(pathToUse, this.clazz);
		}
	}

}
```

### AbstractApplicationContext：抽象应用上下

意味着Spring中的ApplicationContext的所有实现类都拥有加载资源的能力。其直接继承DefaultResourceLoader，且间接的实现ApplicationContext接口，而ApplicationContext继承ResourcePatternResolver接口，具体实现如下：

```java
private ResourcePatternResolver resourcePatternResolver; 

public AbstractApplicationContext() { //构造AbstractApplicationContext对象时直接调用getResourcePatternResolver方法返回PathMatchingResourcePatternResolver
	this.resourcePatternResolver = getResourcePatternResolver();
}

@Override
public Resource[] getResources(String locationPattern) throws IOException {
	return this.resourcePatternResolver.getResources(locationPattern); //使用内部的成员变量直接获取资源
}

protected ResourcePatternResolver getResourcePatternResolver() {
	return new PathMatchingResourcePatternResolver(this);
}

```

从当前代码可看出，Spring框架中默认使用PathMatchingResourcePatternResolver来解析获取所有匹配的资源


### FileSystemResourceLoader：文件系统资源加载器

FileSystemResourceLoader使用FileSystemContextResource来定义获取到的资源对象

```java

/**
 * {@link ResourceLoader} implementation that resolves plain paths as 解析普通路径作为文件系统资源而不是类路径资源
 * file system resources rather than as class path resources
 * (the latter is {@link DefaultResourceLoader}'s default strategy). 后者是(类路径资源)DefaultResourceLoader默认的策略
 *
 * <p><b>NOTE:</b> Plain paths will always be interpreted as relative 普通路径总是被解释为相对于当前VM的工作目录
 * to the current VM working directory, even if they start with a slash.  即使它们以斜杠开头
 * (This is consistent with the semantics in a Servlet container.) 这与Servlet容器中的语义一致
 * <b>Use an explicit "file:" prefix to enforce an absolute file path.</b> 前缀以强制执行绝对文件路径
 *
 * <p>{@link org.springframework.context.support.FileSystemXmlApplicationContext}
 * is a full-fledged(一个成熟的) ApplicationContext implementation that provides
 * the same resource path resolution strategy.
 *
 * @author Juergen Hoeller
 * @since 1.1.3
 * @see DefaultResourceLoader
 * @see org.springframework.context.support.FileSystemXmlApplicationContext
 */
public class FileSystemResourceLoader extends DefaultResourceLoader {

	/**
	 * Resolve resource paths as file system paths. 将资源路径解析为文件系统路径
	 * <p>Note: Even if a given path starts with a slash, it will get
	 * interpreted as relative to the current VM working directory.
	 * @param path the path to the resource
	 * @return the corresponding Resource handle
	 * @see FileSystemResource
	 * @see org.springframework.web.context.support.ServletContextResourceLoader#getResourceByPath
	 */
	@Override
	protected Resource getResourceByPath(String path) {
		if (path.startsWith("/")) { //截取
			path = path.substring(1);
		}
		return new FileSystemContextResource(path);
	}


	/**
	 * FileSystemResource that explicitly expresses a context-relative path
	 * through implementing the ContextResource interface.
	 */
	private static class FileSystemContextResource extends FileSystemResource implements ContextResource {

		public FileSystemContextResource(String path) {
			super(path);
		}

		@Override
		public String getPathWithinContext() {
			return getPath();
		}
	}

}

```

### ServletContextResourceLoader：Servlet上下文资源加载器

ServletContextResourceLoader返回ServletContextResource的资源

```java

/**
 * ResourceLoader implementation that resolves paths as ServletContext 解析作为ServletContext资源
 * resources, for use outside a WebApplicationContext (for example, 在WebApplicationContext之外使用，如HttpServletBean，或者
 * in an HttpServletBean or GenericFilterBean subclass). GenericFilterBean子类
 *
 * <p>Within a WebApplicationContext, resource paths are automatically
 * resolved as ServletContext resources by the context implementation.
 *
 * @author Juergen Hoeller
 * @since 1.0.2
 * @see #getResourceByPath
 * @see ServletContextResource
 * @see org.springframework.web.context.WebApplicationContext
 * @see org.springframework.web.servlet.HttpServletBean
 * @see org.springframework.web.filter.GenericFilterBean
 */
public class ServletContextResourceLoader extends DefaultResourceLoader {

	private final ServletContext servletContext; 


	/**
	 * Create a new ServletContextResourceLoader.
	 * @param servletContext the ServletContext to load resources with
	 */
	public ServletContextResourceLoader(ServletContext servletContext) {
		this.servletContext = servletContext;
	}

	/**
	 * This implementation supports file paths beneath the root of the web application.
	 * @see ServletContextResource
	 */
	@Override
	protected Resource getResourceByPath(String path) {
		return new ServletContextResource(this.servletContext, path);
	}

}

```

## ResourcePatternResolver接口定义

```java
/**
 * Strategy interface for resolving a location pattern (for example,  策略接口，解析一个位置模式为资源对象
 * an Ant-style path pattern) into Resource objects. 
 *
 * <p>This is an extension to the {@link org.springframework.core.io.ResourceLoader}
 * interface. A passed-in ResourceLoader 传递一个ResourceLoader (for example, an  如ApplicationContext、ResourceLoaderAware
 * {@link org.springframework.context.ApplicationContext} passed in via  在上下文中运行时通过ResourceLoader
 * {@link org.springframework.context.ResourceLoaderAware} when running in a context)
 * can be checked whether it implements this extended interface too. 可以检查它是否也实现了这个扩展接口
 *
 * <p>{@link PathMatchingResourcePatternResolver} is a standalone implementation 
 * that is usable outside an ApplicationContext, also used by 可以在ApplicationContext之外使用，也可以在ResourceArrayPropertyEditor使用, 用于填充资源数组bean属性
 * {@link ResourceArrayPropertyEditor} for populating Resource array bean properties. 
 *
 * <p>Can be used with any sort of location pattern (e.g. "/WEB-INF/*-context.xml"): 可以被任意短路径模式使用
 * Input patterns have to match the strategy implementation. This interface just 输入模式必须匹配策略实现
 * specifies the conversion method rather than a specific pattern format. 这个接口只是指定了转换方法而不是特定的模式格式
 *
 * <p>This interface also suggests a new resource prefix "classpath*:" for all 这个接口建议使用新的资源协议"classpath*:"
 * matching resources from the class path. Note that the resource location is 用于从类路径中匹配所有资源
 * expected to be a path without placeholders in this case (e.g. "/beans.xml"); 资源位置是预期的，没有占位符
 * JAR files or classes directories can contain multiple files of the same name. JAR文件或类目录可以包含多个同名文件
 *
 * @author Juergen Hoeller
 * @since 1.0.2
 * @see org.springframework.core.io.Resource
 * @see org.springframework.core.io.ResourceLoader
 * @see org.springframework.context.ApplicationContext
 * @see org.springframework.context.ResourceLoaderAware
 */
public interface ResourcePatternResolver extends ResourceLoader {

	/**
	 * Pseudo URL prefix for all matching resources from the class path: "classpath*:" 伪URL前缀，用于类路径中所有匹配的资源
	 * This differs from ResourceLoader's classpath URL prefix in that it 与ResourceLoader中的classpath前缀不同的是，它检索
	 * retrieves all matching resources for a given name (e.g. "/beans.xml"),给定名称的所有匹配资源
	 * for example in the root of all deployed JAR files. 例如，在所有已部署JAR文件的根目录中
	 * @see org.springframework.core.io.ResourceLoader#CLASSPATH_URL_PREFIX
	 */
	String CLASSPATH_ALL_URL_PREFIX = "classpath*:";

	/**
	 * Resolve the given location pattern into Resource objects. 将给定的位置模式解析为资源对象
	 * <p>Overlapping resource entries that point to the same physical  应该尽可能避免指向相同物理资源的重叠资源项
	 * resource should be avoided, as far as possible. The result should
	 * have set semantics.
	 * @param locationPattern the location pattern to resolve
	 * @return the corresponding Resource objects
	 * @throws IOException in case of I/O errors
	 */
	Resource[] getResources(String locationPattern) throws IOException;

}


```

### PathMatchingResourcePatternResolver：路径匹配资源模式解析器

PathMatchingResourcePatternResolver为ResourcePatternResolver的实现，能够解析一个指定资源位置路径为一个或者多个匹配资源。资源路径可能是一个简单的一对一映射目标资源的路径或者包含指定的"classpath\*:"前缀，内部使用Ant-style风格的通配符，具体匹配过程请参考("https://docs.spring.io/spring/docs/current/javadoc-api/")文档

- 路径中不以"classpath\*:"前缀时，解析器将返回单一的资源，如"file:C:/context.xml"、"classpath:/context.xml"、"/WEB-INF/context.xml"

- 路径中包括Ant-style模式时，如"/WEB-INF/\*-context.xml"、" com/mycompany/\*\*/applicationContext.xml"、"file:C:/some/path/\*-context.xml"、" classpath:com/mycompany/\*\*/applicationContext.xml"，解析器尝试去解析通配符，以获取匹配到的所有资源


```java
/**
 * A {@link ResourcePatternResolver} implementation that is able to resolve a
 * specified resource location path into one or more matching Resources.
 * The source path may be a simple path which has a one-to-one mapping to a
 * target {@link org.springframework.core.io.Resource}, or alternatively
 * may contain the special "{@code classpath*:}" prefix and/or
 * internal Ant-style regular expressions (matched using Spring's  使用spring提供的AntPathMatcher匹配
 * {@link org.springframework.util.AntPathMatcher} utility).
 * Both of the latter are effectively wildcards. ：后者都是有效的通配符
 *
 * <p><b>No Wildcards:</b>
 *
 * <p>In the simple case, if the specified location path does not start with the
 * {@code "classpath*:}" prefix, and does not contain a PathMatcher pattern,
 * this resolver will simply return a single resource via a
 * {@code getResource()} call on the underlying {@code ResourceLoader}.
 * Examples are real URLs such as "{@code file:C:/context.xml}", pseudo-URLs
 * such as "{@code classpath:/context.xml}", and simple unprefixed paths
 * such as "{@code /WEB-INF/context.xml}". The latter will resolve in a
 * fashion specific to the underlying {@code ResourceLoader} (e.g.
 * {@code ServletContextResource} for a {@code WebApplicationContext}).
 *
 * <p><b>Ant-style Patterns:</b>
 *
 * <p>When the path location contains an Ant-style pattern, e.g.:
 * <pre class="code">
 * /WEB-INF/*-context.xml
 * com/mycompany/**&#47;applicationContext.xml
 * file:C:/some/path/*-context.xml
 * classpath:com/mycompany/**&#47;applicationContext.xml</pre>
 * the resolver follows(遵循) a more complex复杂 but defined procedure 定义过程to try to resolve
 * the wildcard. It produces a {@code Resource} for the path up to the last  它为最后没有通配符段路径生成一个Resource
 * non-wildcard segment and obtains a {@code URL} from it. If this URL is 并从中获取一个URL，如果这个URL不是一个jar:URL
 * not a "{@code jar:}" URL or container-specific variant  /'veərɪənt/ (e.g.   或者多样特定容器，如"zip:" "wsjar"
 * "{@code zip:}" in WebLogic, "{@code wsjar}" in WebSphere", etc.), 时，从其中获取一个文件，
 * then a {@code java.io.File} is obtained from it, and used to resolve the  并用于通过遍历文件系统来解析通配符
 * wildcard by walking the filesystem. In the case of a jar URL, the resolver 在一个jar URL情况下，解析器从其中得到一个
 * either gets a {@code java.net.JarURLConnection} from it, or manually parses  JarURLConnection或者手动解析 Jar URL 
 * the jar URL, and then traverses the contents of the jar file, to resolve the  然后遍历jar文件的内容，用于解析这个通配符
 * wildcards.
 *
 * <p><b>Implications on portability:</b> 影响可移植性
 *
 * <p>If the specified path is already a file URL (either explicitly 明确, or 如果指定的路径是一个File URL时，
 * implicitly 暗中 because the base {@code ResourceLoader} is a filesystem one, 因为基础的ResourceLoader是一个文件系统
 * then wildcarding is guaranteed to work in a completely portable fashion. 通配符保证以完全可移植的方式工作
 *
 * <p>If the specified path is a classpath location, then the resolver must 如果指定的路径是一个类路径位置时，解析器必须
 * obtain the last non-wildcard path segment URL via a  获取最后非通配符路径段的URL,通过一个getResource()调用
 * {@code Classloader.getResource()} call. Since this is just a 由于这个仅仅是一个路径节点
 * node of the path (not the file at the end 不是最后的文件) it is actually undefined 它实际上是明确的未定义的
 * (in the ClassLoader Javadocs) exactly what sort(种类) of a URL is returned in URL 
 * this case. In practice, it is usually a {@code java.io.File} representing 实际上，它通常代表一个目录，
 * the directory, where the classpath resource resolves to a filesystem 类路径资源解析为一个文件系统位置或者一个jar URL
 * location, or a jar URL of some sort, where the classpath resource resolves 
 * to a jar location. Still, there is a portability concern on this operation.
 *
 * <p>If a jar URL is obtained for the last non-wildcard segment, the resolver 如果获取了最后一个非通配符段的jar URL 
 * must be able to get a {@code java.net.JarURLConnection} from it, or 解析器必须能够得到一个JarURLConnection，或者
 * manually parse the jar URL, to be able to walk the contents of the jar, 手动解析JAR URL,
 * and resolve the wildcard. This will work in most environments, but will 这将在大多数环境中工作
 * fail in others, and it is strongly recommended that the wildcard 强烈推荐 来自jar的资源的通配符解析在您依赖它之前，要在您的特定环境中进行彻底的测试
 * resolution of resources coming from jars be thoroughly(彻底地) tested in your
 * specific environment before you rely on it.
 *
 * <p><b>{@code classpath*:} Prefix:</b>  classpath*:前缀
 * 
 * <p>There is special support for retrieving multiple class path resources with 特殊支持多类路径资源检索，使用相同的名字
 * the same name, via the "{@code classpath*:}" prefix. For example,
 * "{@code classpath*:META-INF/beans.xml}" will find all "beans.xml"
 * files in the class path, be it in "classes" directories or in JAR files. 在classes目录或者一个JAR文件中
 * This is particularly useful for autodetecting config files of the same name  在每一个jar文件中 
 * at the same location within each jar file. Internally, this happens via a 自动检查在相同位置的相同名字的配置文件非常有用
 * {@code ClassLoader.getResources()} call, and is completely portable.
 *
 * <p>The "classpath*:" prefix can also be combined with a PathMatcher pattern in 也能组合一个带有PathMatcher模式的剩余位置路径
 * the rest of the location path, for example "classpath*:META-INF/*-beans.xml".
 * In this case, the resolution strategy is fairly simple: a  解决策略非常简单，在没有通配符的最后路径阶段调用getResources()
 * {@code ClassLoader.getResources()} call is used on the last non-wildcard  在类加载器的等级上取获取所有匹配的资源
 * path segment to get all the matching resources in the class loader hierarchy,
 * and then off each resource the same PathMatcher resolution strategy described 然后，在每个资源之外，通配符子路径都使用上面描述的相同路径匹配器解析策略
 * above is used for the wildcard subpath.
 *
 * <p><b>Other notes:</b>
 *
 * <p><b>WARNING:</b> Note that "{@code classpath*:}" when combined with
 * Ant-style patterns will only work reliably with at least one root directory 在模式开始之前,只能在至少一个根目录下可靠地工作
 * before the pattern starts, unless the actual target files reside in the file 除非实际目标文件驻留在文件系统
 * system. This means that a pattern like "{@code classpath*:*.xml}" will  这意味着像"classpath*:*.xml"模式不会
 * <i>not</i> retrieve files from the root of jar files but rather only from the  从jar文件的root中检索文件，而只是从
 * root of expanded directories. This originates from a limitation in the JDK's 展开的根目录检索。这源于JDK的制，只会返回
 * locations for a passed-in empty String (indicating potential roots to search 表示要搜索的潜在根). 
 																							文件系统传入一个空字
 * {@code ClassLoader.getResources()} method which only returns file system  ClassLoader.getResources()方法限符串位置，
 * This {@code ResourcePatternResolver} implementation is trying to mitigate  /'mɪtɪgeɪt/ the ResourcePatternResolver实现尝试减轻jar根查找，通过URLClassLoader内省和 "java.class.path" 值 限制
 * jar root lookup limitation through {@link URLClassLoader} introspection and
 * "java.class.path" manifest evaluation 显式求值; however, without portability guarantees. 但是，没有可移植性保证
 *
 * <p><b>WARNING:</b> Ant-style patterns with "classpath:" resources are not 使用"classpath:"的Ant-style模式资源
 * guaranteed to find matching resources if the root package to search is available 不能确保找到的匹配资源是否从根包搜索
 * in multiple class path locations. This is because a resource such as 可获取在多个类路径位置
 * <pre class="code">
 *     com/mycompany/package1/service-context.xml
 * </pre>
 * may be in only one location, but when a path such as
 * <pre class="code">
 *     classpath:com/mycompany/**&#47;service-context.xml
 * </pre>
 * is used to try to resolve it, the resolver will work off the (first) URL 先查找到的返回
 * returned by {@code getResource("com/mycompany");}. If this base package node 如果基础包节点在多个类加载器位置存储
 * exists in multiple classloader locations, the actual end resource may not be 实际的资源可能不在下面
 * underneath. Therefore, preferably, use "{@code classpath*:}" with the same 因此，较好的选择使用相同的ant-style模式 classpath*:，在这种情况下，
 * Ant-style pattern in such a case, which will search <i>all</i> class path 搜索全部类路径包含root包
 * locations that contain the root package.
 *
 * @author Juergen Hoeller
 * @author Colin Sampaleanu
 * @author Marius Bogoevici
 * @author Costin Leau
 * @author Phillip Webb
 * @since 1.0.2
 * @see #CLASSPATH_ALL_URL_PREFIX
 * @see org.springframework.util.AntPathMatcher
 * @see org.springframework.core.io.ResourceLoader#getResource(String)
 * @see ClassLoader#getResources(String)
 */
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {

	private static final Log logger = LogFactory.getLog(PathMatchingResourcePatternResolver.class);

	@Nullable
	private static Method equinoxResolveMethod;

	static {
		try {
			// Detect Equinox OSGi (e.g. on WebSphere 6.1)
			Class<?> fileLocatorClass = ClassUtils.forName("org.eclipse.core.runtime.FileLocator",
					PathMatchingResourcePatternResolver.class.getClassLoader());
			equinoxResolveMethod = fileLocatorClass.getMethod("resolve", URL.class);
			logger.debug("Found Equinox FileLocator for OSGi bundle URL resolution");
		}
		catch (Throwable ex) {
			equinoxResolveMethod = null;
		}
	}


	private final ResourceLoader resourceLoader; //资源加载器

	private PathMatcher pathMatcher = new AntPathMatcher(); //目录解析器


	/**
	 * Create a new PathMatchingResourcePatternResolver with a DefaultResourceLoader. 使用默认的类加载器
	 * <p>ClassLoader access will happen via the thread context class loader.
	 * @see org.springframework.core.io.DefaultResourceLoader
	 */
	public PathMatchingResourcePatternResolver() {
		this.resourceLoader = new DefaultResourceLoader();
	}

	/**
	 * Create a new PathMatchingResourcePatternResolver.
	 * <p>ClassLoader access will happen via the thread context class loader. 类加载器访问将通过线程上下文类加载器进行
	 * @param resourceLoader the ResourceLoader to load root directories and 加载根目录和实际的资源
	 * actual resources with
	 */
	public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
		Assert.notNull(resourceLoader, "ResourceLoader must not be null");
		this.resourceLoader = resourceLoader;
	}

	/**
	 * Create a new PathMatchingResourcePatternResolver with a DefaultResourceLoader.
	 * @param classLoader the ClassLoader to load classpath resources with, 用来加载类路径资源的类加载器
	 * or {@code null} for using the thread context class loader
	 * at the time of actual resource access
	 * @see org.springframework.core.io.DefaultResourceLoader
	 */
	public PathMatchingResourcePatternResolver(@Nullable ClassLoader classLoader) {
		this.resourceLoader = new DefaultResourceLoader(classLoader);
	}


	/**
	 * Return the ResourceLoader that this pattern resolver works with. 返回此模式解析器使用的ResourceLoader
	 */
	public ResourceLoader getResourceLoader() {
		return this.resourceLoader;
	}

	@Override
	@Nullable
	public ClassLoader getClassLoader() {
		return getResourceLoader().getClassLoader();
	}

	/**
	 * Set the PathMatcher implementation to use for this 设置路径匹配器的实现用于这个资源模式解析，默认为AntPathMatcher
	 * resource pattern resolver. Default is AntPathMatcher.
	 * @see org.springframework.util.AntPathMatcher
	 */
	public void setPathMatcher(PathMatcher pathMatcher) {
		Assert.notNull(pathMatcher, "PathMatcher must not be null");
		this.pathMatcher = pathMatcher;
	}

	/**
	 * Return the PathMatcher that this resource pattern resolver uses. 返回此资源模式解析器使用的路径匹配器
	 */
	public PathMatcher getPathMatcher() {
		return this.pathMatcher;
	}


	@Override
	public Resource getResource(String location) {
		return getResourceLoader().getResource(location);
	}

	@Override
	public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) { //路径以"classpath*:"开始
			// a class path resource (multiple resources for same name possible) 类路径资源，多个资源可能具有相同的名称
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) { //是否路径匹配
				// a class path resource pattern 类路径资源模式
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name 所有具有给定名称的类路径资源
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		else {
			// Generally only look for a pattern after a prefix here, 通常只在这里的前缀后面查找模式
			// and on Tomcat only after the "*/" separator for its "war:" protocol. 在Tomcat上，只有在“*/”分隔符之后才使用它的“war:”协议
			int prefixEnd = (locationPattern.startsWith("war:") ? locationPattern.indexOf("*/") + 1 : //截取协议之前后的部分
					locationPattern.indexOf(':') + 1);
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) { 
				// a file pattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// a single resource with the given name 具有给定名称的单个资源
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}

	/**
	 * Find all class location resources with the given location via the ClassLoader. 通过类加载器查找具有给定位置的所有类位置资源
	 * Delegates to {@link #doFindAllClassPathResources(String)}. 委托给
	 * @param location the absolute path within the classpath 类路径中的绝对路径
	 * @return the result as Resource array
	 * @throws IOException in case of I/O errors
	 * @see java.lang.ClassLoader#getResources
	 * @see #convertClassLoaderURL
	 */
	protected Resource[] findAllClassPathResources(String location) throws IOException {
		String path = location;
		if (path.startsWith("/")) { //截取"/"
			path = path.substring(1);
		}
		Set<Resource> result = doFindAllClassPathResources(path); //委托，返回set集合
		if (logger.isDebugEnabled()) {
			logger.debug("Resolved classpath location [" + location + "] to resources " + result);
		}
		return result.toArray(new Resource[0]);
	}

	/**
	 * Find all class location resources with the given path via the ClassLoader.
	 * Called by {@link #findAllClassPathResources(String)}.
	 * @param path the absolute path within the classpath (never a leading slash)类路径中的绝对路径(没有"/")
	 * @return a mutable Set of matching Resource instances 可变的匹配资源实例集
	 * @since 4.1.1
	 */
	protected Set<Resource> doFindAllClassPathResources(String path) throws IOException {
		Set<Resource> result = new LinkedHashSet<>(16);
		ClassLoader cl = getClassLoader();
		//类加载器不为null时使用其加载资源，否则使用ClassLoader根据path获取系统资源
		Enumeration<URL> resourceUrls = (cl != null ? cl.getResources(path) : ClassLoader.getSystemResources(path));
		while (resourceUrls.hasMoreElements()) {
			URL url = resourceUrls.nextElement();
			result.add(convertClassLoaderURL(url)); //将URL转换为UrlResource资源
		}
		if ("".equals(path)) { 
			// The above result is likely to be incomplete 上述结果可能是不完整的, i.e. only containing file system references. 只包含文件系统引用,还需要指向每一个jar文件的类路径
			// We need to have pointers to each of the jar files on the classpath as well...
			addAllClassLoaderJarRoots(cl, result);
		}
		return result;
	}

	/**
	 * Convert the given URL as returned from the ClassLoader into a {@link Resource}. 将从类加载器返回的给定URL转换为Resource
	 * <p>The default implementation simply creates a {@link UrlResource} instance.
	 * @param url a URL as returned from the ClassLoader
	 * @return the corresponding Resource object
	 * @see java.lang.ClassLoader#getResources
	 * @see org.springframework.core.io.Resource
	 */
	protected Resource convertClassLoaderURL(URL url) {
		return new UrlResource(url);
	}

	/**
	 * Search all {@link URLClassLoader} URLs for jar file references and add them to the 搜索所有的jar文件引用的url，添加
	 * given set of resources in the form of pointers to the root of the jar file content. 到给定的set集合资源，以指针的形式
	 * @param classLoader the ClassLoader to search (including its ancestors)  指向jar文件内容的根目录 
	 * @param result the set of resources to add jar roots to
	 * @since 4.1.1
	 */
	protected void addAllClassLoaderJarRoots(@Nullable ClassLoader classLoader, Set<Resource> result) {
		if (classLoader instanceof URLClassLoader) { 
			try {
				for (URL url : ((URLClassLoader) classLoader).getURLs()) {
					try { //使用UrlResource加载jar中的根路径资源
						UrlResource jarResource = new UrlResource(
								ResourceUtils.JAR_URL_PREFIX + url + ResourceUtils.JAR_URL_SEPARATOR);
						if (jarResource.exists()) { //jar文件存在，添加到返回集合中
							result.add(jarResource);
						}
					}
					catch (MalformedURLException ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Cannot search for matching files underneath [" + url +
									"] because it cannot be converted to a valid 'jar:' URL: " + ex.getMessage());
						}
					}
				}
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Cannot introspect jar files since ClassLoader [" + classLoader +
							"] does not support 'getURLs()': " + ex);
				}
			}
		}

		//类加载器等于默认加载器
		if (classLoader == ClassLoader.getSystemClassLoader()) { 
			// "java.class.path" manifest evaluation... 
			addClassPathManifestEntries(result);
		}

		if (classLoader != null) {
			try {
				// Hierarchy traversal... 层次遍历
				addAllClassLoaderJarRoots(classLoader.getParent(), result);
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Cannot introspect jar files in parent ClassLoader since [" + classLoader +
							"] does not support 'getParent()': " + ex);
				}
			}
		}
	}

	/**
	 * Determine jar file references from the "java.class.path." manifest property and add them 确定jar文件引用来自 
	 			java.class.path. 属性清单
	 * to the given set of resources in the form of pointers to the root of the jar file content. 以指向jar文件内容根的指针的形式
	 * @param result the set of resources to add jar roots to
	 * @since 4.3
	 */
	protected void addClassPathManifestEntries(Set<Resource> result) {
		try {
			String javaClassPathProperty = System.getProperty("java.class.path"); //获取class.path属性，
			for (String path : StringUtils.delimitedListToStringArray( //分隔符截取路径
					javaClassPathProperty, System.getProperty("path.separator"))) {
				try {
					String filePath = new File(path).getAbsolutePath(); //获取绝对路径
					int prefixIndex = filePath.indexOf(':');
					if (prefixIndex == 1) {
						// Possibly "c:" drive prefix on Windows 驱动器前缀, to be upper-cased for proper duplicate detection
						filePath = StringUtils.capitalize(filePath); //首字母大写
					}
					UrlResource jarResource = new UrlResource(ResourceUtils.JAR_URL_PREFIX +
							ResourceUtils.FILE_URL_PREFIX + filePath + ResourceUtils.JAR_URL_SEPARATOR);
					// Potentially overlapping with URLClassLoader.getURLs() result above! //去重
					if (!result.contains(jarResource) && !hasDuplicate(filePath, result) && jarResource.exists()) {
						result.add(jarResource); //添加
					}
				}
				catch (MalformedURLException ex) {
					if (logger.isDebugEnabled()) {
						logger.debug("Cannot search for matching files underneath [" + path +
								"] because it cannot be converted to a valid 'jar:' URL: " + ex.getMessage());
					}
				}
			}
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to evaluate 'java.class.path' manifest entries: " + ex);
			}
		}
	}

	/**
	 * Check whether the given file path has a duplicate but differently structured entry 检查给定的文件路径是否有副本
	 * in the existing result, i.e. with or without a leading slash. 但结构不同的条目。"有或没有前导斜杠"
	 * @param filePath the file path (with or without a leading slash) 文件路径，包括或者不包括"/"
	 * @param result the current result
	 * @return {@code true} if there is a duplicate (i.e. to ignore the given file path),
	 * {@code false} to proceed with adding a corresponding resource to the current result
	 */
	private boolean hasDuplicate(String filePath, Set<Resource> result) {
		if (result.isEmpty()) {
			return false;
		}
		//文件路径以"/"开始时，截取 否则添加"/"
		String duplicatePath = (filePath.startsWith("/") ? filePath.substring(1) : "/" + filePath);
		try {
			return result.contains(new UrlResource(ResourceUtils.JAR_URL_PREFIX + ResourceUtils.FILE_URL_PREFIX +
					duplicatePath + ResourceUtils.JAR_URL_SEPARATOR));
		}
		catch (MalformedURLException ex) {
			// Ignore: just for testing against duplicate.
			return false;
		}
	}

	/**
	 * Find all resources that match the given location pattern via the 通过ant路径匹配器找到所有匹配给定位置的资源
	 * Ant-style PathMatcher. Supports resources in jar files and zip files，支持在jar文件和zip文件和文件系统
	 * and in the file system.
	 * @param locationPattern the location pattern to match
	 * @return the result as Resource array
	 * @throws IOException in case of I/O errors
	 * @see #doFindPathMatchingJarResources
	 * @see #doFindPathMatchingFileResources
	 * @see org.springframework.util.PathMatcher
	 */
	protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
		String rootDirPath = determineRootDir(locationPattern);  //获取位置目录的根目录，如"classpath*:META-INF/*.txt" 返回classpath*:META-INF/
		String subPattern = locationPattern.substring(rootDirPath.length()); //从locationPattern中截取子模式("*.txt")
		Resource[] rootDirResources = getResources(rootDirPath); //获取根目录资源，此时会调用 findAllClassPathResources 获取所有类路径资源
		Set<Resource> result = new LinkedHashSet<>(16); 
		for (Resource rootDirResource : rootDirResources) { //遍历根目录资源
			rootDirResource = resolveRootDirResource(rootDirResource); //解析根目录资源，默认实现直接返回原始资源
			URL rootDirUrl = rootDirResource.getURL(); //获取资源的URL
			if (equinoxResolveMethod != null && rootDirUrl.getProtocol().startsWith("bundle")) { //OSGi资源
				URL resolvedUrl = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirUrl);
				if (resolvedUrl != null) {
					rootDirUrl = resolvedUrl;
				}
				rootDirResource = new UrlResource(rootDirUrl);
			}
			if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) { //vfs资源
				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
			}
			else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) { //jar资源 协议"jar", "war, ""zip", "vfszip" or "wsjar"
				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
			}
			else { //其他，查找路径匹配文件资源
				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
			}
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Resolved location pattern [" + locationPattern + "] to resources " + result);
		}
		return result.toArray(new Resource[0]);
	}

	/**
	 * Determine the root directory for the given location. 确定给定位置的根目录
	 * <p>Used for determining the starting point for file matching, 用于确定文件匹配的起始点
	 * resolving the root directory location to a {@code java.io.File} 解析根目录位置
	 * and passing it into {@code retrieveMatchingFiles}, with the 传递给retrieveMatchingFiles 
	 * remainder of the location as pattern. 位置的其余部分作为模式
	 * <p>Will return "/WEB-INF/" for the pattern "/WEB-INF/*.xml", 
	 * for example.
	 * @param location the location to check
	 * @return the part of the location that denotes the root directory
	 * @see #retrieveMatchingFiles
	 */
	protected String determineRootDir(String location) {  //location="classpath*:META-INF/*.txt"
		int prefixEnd = location.indexOf(':') + 1; //前缀结束 (META-INF/*.txt 前缀)
		int rootDirEnd = location.length(); //根位置长度
		//循环获取根目录结束位置，且匹配当前路径匹配器
		while (rootDirEnd > prefixEnd && getPathMatcher().isPattern(location.substring(prefixEnd, rootDirEnd))) {
			rootDirEnd = location.lastIndexOf('/', rootDirEnd - 2) + 1; "/"结尾，META-INF + 1 
		}
		if (rootDirEnd == 0) {
			rootDirEnd = prefixEnd;
		}
		return location.substring(0, rootDirEnd);
	}

	/**
	 * Resolve the specified resource for path matching. 为路径匹配解析指定的资源
	 * <p>By default, Equinox OSGi "bundleresource:" / "bundleentry:" URL will be
	 * resolved into a standard jar file URL that be traversed using Spring's
	 * standard jar file traversal algorithm 遍历算法. For any preceding custom resolution, 用于任何以前的自定义解析
	 * override this method and replace the resource handle accordingly. 重写此方法并相应地替换资源句柄
	 * @param original the resource to resolve
	 * @return the resolved resource (may be identical to the passed-in resource)
	 * @throws IOException in case of resolution failure
	 */
	protected Resource resolveRootDirResource(Resource original) throws IOException {
		return original;
	}

	/**
	 * Return whether the given resource handle indicates a jar resource 返回给定的资源句柄是否为一个jar资源
	 * that the {@code doFindPathMatchingJarResources} method can handle. doFindPathMatchingJarResources方法处理
	 * <p>By default, the URL protocols "jar", "zip", "vfszip and "wsjar" 默认URL协议"jar", "zip", "vfszip and "wsjar" 被视为jar资源
	 * will be treated as jar resources. This template method allows for 这个模板方法允许进一步检测其他像jar资源的类型
	 * detecting further kinds of jar-like resources, e.g. through
	 * {@code instanceof} checks on the resource handle type.
	 * @param resource the resource handle to check
	 * (usually the root directory to start path matching from) 通常从根目录开始路径匹配
	 * @see #doFindPathMatchingJarResources
	 * @see org.springframework.util.ResourceUtils#isJarURL
	 */
	protected boolean isJarResource(Resource resource) throws IOException {
		return false;
	}

	/**
	 * Find all resources in jar files that match the given location pattern 在jar文件中查找匹配给定位置模式的所有资源
	 * via the Ant-style PathMatcher.
	 * @param rootDirResource the root directory as Resource 根目录作为资源(jar:file:/D:/IntelliJ%20IDEA/lib/idea_rt.jar!/META-INF/)
	 * @param rootDirURL the pre-resolved root directory URL 预解析的根目录URL
	 * @param subPattern the sub pattern to match (below the root directory)
	 * @return a mutable Set of matching Resource instances
	 * @throws IOException in case of I/O errors
	 * @since 4.3
	 * @see java.net.JarURLConnection
	 * @see org.springframework.util.PathMatcher
	 */
	protected Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirURL, String subPattern)
			throws IOException {

		URLConnection con = rootDirURL.openConnection(); //根路径URL,打开连接
		JarFile jarFile; //jar文件
		String jarFileUrl; //jar文件url路径
		String rootEntryPath; //根实体路径
		boolean closeJarFile; //是否关闭jar文件

		if (con instanceof JarURLConnection) { 
			// Should usually be the case for traditional JAR files. 传统JAR文件通常应该是这种情况
			JarURLConnection jarCon = (JarURLConnection) con;
			ResourceUtils.useCachesIfNecessary(jarCon);
			jarFile = jarCon.getJarFile(); //获取jar文件
			jarFileUrl = jarCon.getJarFileURL().toExternalForm(); //file:/D:/IntelliJ%20IDEA/lib/idea_rt.jar ,外部形态
			JarEntry jarEntry = jarCon.getJarEntry(); //jar文件中某个文件或者目录实体
			rootEntryPath = (jarEntry != null ? jarEntry.getName() : ""); //根目录null时为""
			closeJarFile = !jarCon.getUseCaches();
		}
		else {
			// No JarURLConnection -> need to resort to URL file parsing. 不是JarURLConnection，需要求助于URL文件解析
			// We'll assume URLs of the format "jar:path!/entry", with the protocol 假设url的格式通过任意协议
			// being arbitrary as long as following the entry format. 只要遵循输入格式即可
			// We'll also handle paths with and without leading "file:" prefix.
			String urlFile = rootDirURL.getFile();
			try {
				int separatorIndex = urlFile.indexOf(ResourceUtils.WAR_URL_SEPARATOR); //*/
				if (separatorIndex == -1) { //不存在
					separatorIndex = urlFile.indexOf(ResourceUtils.JAR_URL_SEPARATOR); // !/
				}
				if (separatorIndex != -1) { //存在
					jarFileUrl = urlFile.substring(0, separatorIndex);
					rootEntryPath = urlFile.substring(separatorIndex + 2);  // both separators are 2 chars
					jarFile = getJarFile(jarFileUrl); //从jar url中解析JarFile
				}
				else { //不存在
					jarFile = new JarFile(urlFile); //创建JarFile
					jarFileUrl = urlFile;
					rootEntryPath = "";
				}
				closeJarFile = true;
			}
			catch (ZipException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping invalid jar classpath entry [" + urlFile + "]");
				}
				return Collections.emptySet();
			}
		}

		try {
			if (logger.isDebugEnabled()) {
				logger.debug("Looking for matching resources in jar file [" + jarFileUrl + "]");
			}
			if (!"".equals(rootEntryPath) && !rootEntryPath.endsWith("/")) { // 根路径不为"",且不以"/"结尾时，添加后缀
				// Root entry path must end with slash to allow for proper matching. 根条目路径必须以斜杠结束，以便进行适当的匹配
				// The Sun JRE does not return a slash here, but BEA JRockit does. Sun JRE在这里不返回斜杠
				rootEntryPath = rootEntryPath + "/"; 
			}
			Set<Resource> result = new LinkedHashSet<>(8);
			for (Enumeration<JarEntry> entries = jarFile.entries(); entries.hasMoreElements();) { //遍历根目录的jar实体
				JarEntry entry = entries.nextElement();
				String entryPath = entry.getName();
				if (entryPath.startsWith(rootEntryPath)) { //以根目录开始，如果匹配当前路径模式，创建根目录资源的相对资源
					String relativePath = entryPath.substring(rootEntryPath.length());
					if (getPathMatcher().match(subPattern, relativePath)) {
						result.add(rootDirResource.createRelative(relativePath));
					}
				}
			}
			return result; 
		}
		finally {
			if (closeJarFile) {
				jarFile.close();
			}
		}
	}

	/**
	 * Resolve the given jar file URL into a JarFile object. 将给定的jar文件URL解析为JarFile对象
	 */
	protected JarFile getJarFile(String jarFileUrl) throws IOException {
		if (jarFileUrl.startsWith(ResourceUtils.FILE_URL_PREFIX)) { //file:
			try {
				return new JarFile(ResourceUtils.toURI(jarFileUrl).getSchemeSpecificPart());
			}
			catch (URISyntaxException ex) {
				// Fallback for URLs that are not valid URIs (should hardly ever happen).
				return new JarFile(jarFileUrl.substring(ResourceUtils.FILE_URL_PREFIX.length()));
			}
		}
		else {
			return new JarFile(jarFileUrl);
		}
	}

	/**
	 * Find all resources in the file system that match the given location pattern 查找文件系统中与给定位置模式匹配的所有资源
	 * via the Ant-style PathMatcher. ：通过ant样式的路径匹配器
	 * @param rootDirResource the root directory as Resource
	 * @param subPattern the sub pattern to match (below the root directory)
	 * @return a mutable Set of matching Resource instances
	 * @throws IOException in case of I/O errors
	 * @see #retrieveMatchingFiles
	 * @see org.springframework.util.PathMatcher
	 */
	protected Set<Resource> doFindPathMatchingFileResources(Resource rootDirResource, String subPattern)
			throws IOException {

		File rootDir; //获取根目录资源的File的绝对路径
		try {
			rootDir = rootDirResource.getFile().getAbsoluteFile();
		}
		catch (FileNotFoundException ex) {
			if (logger.isInfoEnabled()) {
				logger.info("Cannot search for matching files underneath " + rootDirResource +
						" in the file system: " + ex.getMessage());
			}
			return Collections.emptySet();
		}
		catch (Exception ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Failed to resolve " + rootDirResource + " in the file system: " + ex);
			}
			return Collections.emptySet();
		}
		return doFindMatchingFileSystemResources(rootDir, subPattern); //执行查找匹配的文件系统资源
	}

	/**
	 * Find all resources in the file system that match the given location pattern 在文件系统中，通过 Ant-style PathMatcher
	 * via the Ant-style PathMatcher. 找出所有匹配给定位置模式
	 * @param rootDir the root directory in the file system 文件系统中的根目录
	 * @param subPattern the sub pattern to match (below the root directory) 在根目录下，要匹配的子模式
	 * @return a mutable Set of matching Resource instances
	 * @throws IOException in case of I/O errors
	 * @see #retrieveMatchingFiles
	 * @see org.springframework.util.PathMatcher
	 */
	protected Set<Resource> doFindMatchingFileSystemResources(File rootDir, String subPattern) throws IOException {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for matching resources in directory tree [" + rootDir.getPath() + "]");
		}
		Set<File> matchingFiles = retrieveMatchingFiles(rootDir, subPattern); //检索匹配的文件
		Set<Resource> result = new LinkedHashSet<>(matchingFiles.size());
		for (File file : matchingFiles) {
			result.add(new FileSystemResource(file)); //将匹配的File封装为FileSystemResource文件系统资源对象
		}
		return result;
	}

	/**
	 * Retrieve files that match the given path pattern, 检索与给定路径模式匹配的文件，检查给定目录及其子目录
	 * checking the given directory and its subdirectories.
	 * @param rootDir the directory to start from 开始目录
	 * @param pattern the pattern to match against, 要匹配的模式
	 * relative to the root directory 相对于根目录
	 * @return a mutable Set of matching Resource instances 一组可变的匹配资源实例
	 * @throws IOException if directory contents could not be retrieved
	 */
	protected Set<File> retrieveMatchingFiles(File rootDir, String pattern) throws IOException {
		if (!rootDir.exists()) { //根路径不存在时返回空
			// Silently skip non-existing directories.
			if (logger.isDebugEnabled()) {
				logger.debug("Skipping [" + rootDir.getAbsolutePath() + "] because it does not exist");
			}
			return Collections.emptySet();
		}
		if (!rootDir.isDirectory()) {  //根路径不是目录时返回空
			// Complain louder if it exists but is no directory.
			if (logger.isWarnEnabled()) {
				logger.warn("Skipping [" + rootDir.getAbsolutePath() + "] because it does not denote a directory");
			}
			return Collections.emptySet();
		}
		if (!rootDir.canRead()) { //根路径不能读取时返回空
			if (logger.isWarnEnabled()) {
				logger.warn("Cannot search for matching files underneath directory [" + rootDir.getAbsolutePath() +
						"] because the application is not allowed to read the directory");
			}
			return Collections.emptySet();
		}
		String fullPattern = StringUtils.replace(rootDir.getAbsolutePath(), File.separator, "/");
		if (!pattern.startsWith("/")) { //模式前缀不为"/"，全模式加"/"
			fullPattern += "/";
		}
		fullPattern = fullPattern + StringUtils.replace(pattern, File.separator, "/");
		Set<File> result = new LinkedHashSet<>(8);
		doRetrieveMatchingFiles(fullPattern, rootDir, result); //执行检索匹配的文件
		return result;
	}

	/**
	 * Recursively retrieve files that match the given pattern, 递归检索与给定模式匹配的文件
	 * adding them to the given result list. 将它们添加到给定的结果列表中
	 * @param fullPattern the pattern to match against, 要匹配的模式,使用预写的根目录路径
	 * with prepended root directory path
	 * @param dir the current directory
	 * @param result the Set of matching File instances to add to
	 * @throws IOException if directory contents could not be retrieved
	 */
	protected void doRetrieveMatchingFiles(String fullPattern, File dir, Set<File> result) throws IOException {
		if (logger.isDebugEnabled()) {
			logger.debug("Searching directory [" + dir.getAbsolutePath() +
					"] for files matching pattern [" + fullPattern + "]");
		}
		File[] dirContents = dir.listFiles(); //目录下的文件
		if (dirContents == null) {
			if (logger.isWarnEnabled()) {
				logger.warn("Could not retrieve contents of directory [" + dir.getAbsolutePath() + "]");
			}
			return;
		}
		Arrays.sort(dirContents);//排查
		for (File content : dirContents) {
			String currPath = StringUtils.replace(content.getAbsolutePath(), File.separator, "/");
			if (content.isDirectory() && getPathMatcher().matchStart(fullPattern, currPath + "/")) { //目录时递归查找
				if (!content.canRead()) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping subdirectory [" + dir.getAbsolutePath() +
								"] because the application is not allowed to read the directory");
					}
				}
				else {
					doRetrieveMatchingFiles(fullPattern, content, result);
				}
			}
			if (getPathMatcher().match(fullPattern, currPath)) { //匹配当前路径时将路径添加到当前结果
				result.add(content);
			}
		}
	}


	/**
	 * Inner delegate class, avoiding a hard JBoss VFS API dependency at runtime.
	 */
	private static class VfsResourceMatchingDelegate {

		public static Set<Resource> findMatchingResources(
				URL rootDirURL, String locationPattern, PathMatcher pathMatcher) throws IOException {

			Object root = VfsPatternUtils.findRoot(rootDirURL);
			PatternVirtualFileVisitor visitor =
					new PatternVirtualFileVisitor(VfsPatternUtils.getPath(root), locationPattern, pathMatcher);
			VfsPatternUtils.visit(root, visitor);
			return visitor.getResources();
		}
	}


	/**
	 * VFS visitor for path matching purposes.
	 */
	@SuppressWarnings("unused")
	private static class PatternVirtualFileVisitor implements InvocationHandler {

		private final String subPattern;

		private final PathMatcher pathMatcher;

		private final String rootPath;

		private final Set<Resource> resources = new LinkedHashSet<>();

		public PatternVirtualFileVisitor(String rootPath, String subPattern, PathMatcher pathMatcher) {
			this.subPattern = subPattern;
			this.pathMatcher = pathMatcher;
			this.rootPath = (rootPath.isEmpty() || rootPath.endsWith("/") ? rootPath : rootPath + "/");
		}

		@Override
		@Nullable
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			String methodName = method.getName();
			if (Object.class == method.getDeclaringClass()) {
				if (methodName.equals("equals")) {
					// Only consider equal when proxies are identical.
					return (proxy == args[0]);
				}
				else if (methodName.equals("hashCode")) {
					return System.identityHashCode(proxy);
				}
			}
			else if ("getAttributes".equals(methodName)) {
				return getAttributes();
			}
			else if ("visit".equals(methodName)) {
				visit(args[0]);
				return null;
			}
			else if ("toString".equals(methodName)) {
				return toString();
			}

			throw new IllegalStateException("Unexpected method invocation: " + method);
		}

		public void visit(Object vfsResource) {
			if (this.pathMatcher.match(this.subPattern,
					VfsPatternUtils.getPath(vfsResource).substring(this.rootPath.length()))) {
				this.resources.add(new VfsResource(vfsResource));
			}
		}

		@Nullable
		public Object getAttributes() {
			return VfsPatternUtils.getVisitorAttributes();
		}

		public Set<Resource> getResources() {
			return this.resources;
		}

		public int size() {
			return this.resources.size();
		}

		@Override
		public String toString() {
			return "sub-pattern: " + this.subPattern + ", resources: " + this.resources;
		}
	}

}

```




