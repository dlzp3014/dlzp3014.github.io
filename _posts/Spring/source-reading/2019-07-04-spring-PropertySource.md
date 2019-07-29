---
layout: post
title:  "Spring源码-PropertySource<T>属性源抽象类"
date:   2019-07-04 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

PropertySource<T\> 代表一个键值对的属性源，T代表具体的属性源对象。针对不同的属于源Spring框架提供了不同的实现，类结构如下：
![](/img/post.img/spring/PropertySource.png)







## PropertySource<T>：属性源抽象基类，T可以为任意封装的properties类型

PropertySource对象一般不单独使用，而是通过聚合了多个PropertySource对象的PropertySources对象并结合PropertyResolver属性解析器的实现【具体说明见：[Spring源码-PropertyResolver属性解析器接口、Environment环境信息接口](/2019/07/04/spring-PropertyResolver/)】，基于优先的搜索PropertySources。PropertySource的唯一标识为name，这对于操作上下文的PropertySource非常有用，且可以使用@Configuration和@PropertySource注解添加额外的属性配置文件到Environment环境中【具体实现参考：[Spring源码-@PropertySource注解](/2019/06/24/spring-@PropertySourcer/)】。


- 类描述

```java
/**
 * Abstract base class representing a source of name/value property pairs. The underlying
 * {@linkplain #getSource() source object} may be of any type {@code T} that encapsulates 封装
 * properties. Examples include {@link java.util.Properties} objects, {@link java.util.Map}
 * objects, {@code ServletContext} and {@code ServletConfig} objects (for access to init
 * parameters). Explore the {@code PropertySource} type hierarchy 类层次 to see provided
 * implementations. 
 *
 * <p>{@code PropertySource} objects are not typically used in isolation 对象通常不单独使用, but rather
 * through a {@link PropertySources} object, which aggregates 聚合 property sources and in
 * conjunction with a {@link PropertyResolver} implementation that can perform
 * precedence-based searches across the set of {@code PropertySources}.
 *
 * <p>{@code PropertySource} identity 唯一标识 is determined not based on the content of
 * encapsulated properties, but rather based on the {@link #getName() name} of the
 * {@code PropertySource} alone. This is useful for manipulating {@code PropertySource}
 * objects when in collection contexts. See operations in {@link MutablePropertySources}
 * as well as the {@link #named(String)} and {@link #toString()} methods for details.
 *
 * <p>Note that when working with @{@link
 * org.springframework.context.annotation.Configuration Configuration} classes that
 * the @{@link org.springframework.context.annotation.PropertySource PropertySource}
 * annotation provides a convenient and declarative way 提供便捷、声明的方式 of adding property sources to the
 * enclosing 封闭 {@code Environment}.
 *
 * @since 3.1
 * @see PropertySources
 * @see PropertyResolver
 * @see PropertySourcesPropertyResolver
 * @see MutablePropertySources
 * @see org.springframework.context.annotation.PropertySource
 */

```

- 类实现

```java
public abstract class PropertySource<T> {

	protected final Log logger = LogFactory.getLog(getClass());

	protected final String name; //属性源唯一标识

	protected final T source; //具体的属性源


	/**
	 * Create a new {@code PropertySource} with the given name and source object.
	 */
	public PropertySource(String name, T source) {
		Assert.hasText(name, "Property source name must contain at least one character");
		Assert.notNull(source, "Property source must not be null");
		this.name = name;
		this.source = source;
	}

	/**
	 * Create a new {@code PropertySource} with the given name and with a new
	 * {@code Object} instance as the underlying source.
	 * <p>Often useful in testing scenarios when creating anonymous implementations
	 * that never query an actual source but rather return hard-coded values.
	 */
	@SuppressWarnings("unchecked")
	public PropertySource(String name) {
		this(name, (T) new Object());
	}


	/**
	 * Return the name of this {@code PropertySource}
	 */
	public String getName() {
		return this.name;
	}

	/**
	 * Return the underlying source object for this {@code PropertySource}. 返回底层属性源对象
	 */
	public T getSource() {
		return this.source;
	}

	/**
	 * Return whether this {@code PropertySource} contains the given name.
	 * <p>This implementation simply checks for a {@code null} return value
	 * from {@link #getProperty(String)}. Subclasses may wish to implement
	 * a more efficient algorithm if possible. 子类有可能提供一个更有效算法
	 * @param name the property name to find
	 */
	public boolean containsProperty(String name) {
		return (getProperty(name) != null);
	}

	/**
	 * Return the value associated with the given name, 返回与给定名称关联的值
	 * or {@code null} if not found.
	 * @param name the property to find
	 * @see PropertyResolver#getRequiredProperty(String)
	 */
	@Nullable
	public abstract Object getProperty(String name); 抽象方法：由子类提供

	/**
	 * Produce concise output (type and name) if the current log level does not include
	 * debug. If debug is enabled, produce verbose output including the hash code of the
	 * PropertySource instance and every name/value property pair.
	 * <p>This variable verbosity is useful as a property source such as system properties
	 * or environment variables may contain an arbitrary number of property pairs,
	 * potentially leading to difficult to read exception and log messages.
	 * @see Log#isDebugEnabled()
	 */
	@Override
	public String toString() {
		if (logger.isDebugEnabled()) {
			return getClass().getSimpleName() + "@" + System.identityHashCode(this) +
					" {name='" + this.name + "', properties=" + this.source + "}";
		}
		else {
			return getClass().getSimpleName() + " {name='" + this.name + "'}";
		}
	}


	/**
	 * Return a {@code PropertySource} implementation intended for collection comparison purposes only 收集比较用途.
	 * <p>Primarily for internal use, but given a collection of {@code PropertySource} objects, may be
	 * used as follows:
	 * <pre class="code">
	 * {@code List<PropertySource<?>> sources = new ArrayList<PropertySource<?>>();
	 * sources.add(new MapPropertySource("sourceA", mapA));
	 * sources.add(new MapPropertySource("sourceB", mapB));
	 * assert sources.contains(PropertySource.named("sourceA"));
	 * assert sources.contains(PropertySource.named("sourceB"));
	 * assert !sources.contains(PropertySource.named("sourceC"));
	 * }</pre>
	 * The returned {@code PropertySource} will throw {@code UnsupportedOperationException}
	 * if any methods other than {@code equals(Object)}, {@code hashCode()}, and {@code toString()}
	 * are called.
	 * @param name the name of the comparison {@code PropertySource} to be created and returned. 
	 */
	public static PropertySource<?> named(String name) { //根据name返回一个用于比较的PropertySource，除比较相关方法外，其他方法都不支持操作
		return new ComparisonPropertySource(name);
	}


	/**
	 * {@code PropertySource} to be used as a placeholder in cases where an actual PropertySource被用作一个占位符的场景，在应用上下文件创建时，真实的属性源不能迫切的初始化
	 * property source cannot be eagerly initialized at application context
	 * creation time.  For example, a {@code ServletContext}-based property source 如ServletContext属性源必须等到
	 * must wait until the {@code ServletContext} object is available to its enclosing ApplicationContext包含ServletContext对象可用
	 * {@code ApplicationContext}.  In such cases, a stub should be used to hold the StubPropertySource被用作持有一个
	 * intended default position/order of the property source, then be replaced 故意默认位置/排序的属性源
	 * during context refresh. 在上下文刷新时替换
	 * @see org.springframework.context.support.AbstractApplicationContext#initPropertySources()
	 * @see org.springframework.web.context.support.StandardServletEnvironment
	 * @see org.springframework.web.context.support.ServletContextPropertySource
	 */
	public static class StubPropertySource extends PropertySource<Object> {  占位符属性源

		public StubPropertySource(String name) { //仅包含name
			super(name, new Object());
		}

		/**
		 * Always returns {@code null}.
		 */
		@Override
		@Nullable
		public String getProperty(String name) {
			return null;
		}
	}


	/**
	 * @see PropertySource#named(String)
	 */
	static class ComparisonPropertySource extends StubPropertySource { 用于比较的属性源

		private static final String USAGE_ERROR =
				"ComparisonPropertySource instances are for use with collection comparison only";

		public ComparisonPropertySource(String name) {
			super(name);
		}

		@Override
		public Object getSource() {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}

		@Override
		public boolean containsProperty(String name) {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}

		@Override
		@Nullable
		public String getProperty(String name) {
			throw new UnsupportedOperationException(USAGE_ERROR);
		}
	}

}

```

## EnumerablePropertySource<T>:可列举的属性源

```java
/**
 * A {@link PropertySource} implementation capable of interrogating /in'terəgeitiŋ/询问 its
 * underlying source object to enumerate  all possible property name/value 列举所有可能的name/value对象
 * pairs. Exposes the {@link #getPropertyNames()} method to allow callers 暴露getPropertyNames()方法，运行调用者内省可
 * to introspect available properties without having to access the underlying 用的属性，而不必访问底层源对象
 * source object. This also facilitates 有利于 a more efficient implementation of
 * {@link #containsProperty(String)}, in that it can call {@link #getPropertyNames()}
 * and iterate through the returned array rather than attempting a call to
 * {@link #getProperty(String)} which may be more expensive 昂贵的. Implementations may
 * consider caching the result of {@link #getPropertyNames()} to fully exploit this
 * performance opportunity. 充分利用表现机会
 *
 * <p>Most framework-provided {@code PropertySource} implementations are enumerable; 多数框架提供的PropertySource实现
 * a counter-example 一个相反的例子 would be {@code JndiPropertySource} where, due to the 是可列举的
 * nature of JNDI it is not possible to determine all possible property names at
 * any given time; rather it is only possible to try to access a property
 * (via {@link #getProperty(String)}) in order to evaluate whether it is present
 * or not.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 */
public abstract class EnumerablePropertySource<T> extends PropertySource<T> {

	public EnumerablePropertySource(String name, T source) {
		super(name, source);
	}

	protected EnumerablePropertySource(String name) {
		super(name);
	}


	/**
	 * Return whether this {@code PropertySource} contains a property with the given name.
	 * <p>This implementation checks for the presence of the given name within the
	 * {@link #getPropertyNames()} array.
	 * @param name the name of the property to find
	 */
	@Override
	public boolean containsProperty(String name) {
		return ObjectUtils.containsElement(getPropertyNames(), name);
	}

	/**
	 * Return the names of all properties contained by the 返回所有属性的name
	 * {@linkplain #getSource() source} object (never {@code null}).
	 */
	public abstract String[] getPropertyNames();//模板方法：由具体的子类提供

}
```

## CommandLinePropertySource<T>：命令行属性源

抽象基类，支持命令行参数，T代表命令行参数选项源。具体实现有SimpleCommandLinePropertySource和JOptCommandLinePropertySource，T分别代表CommandLineArgs和OptionSet，详细说明见[Spring源码-CommandLinePropertySource<T\>命令行属性源]()


## CompositePropertySource：组合属性源

组合多个PropertySource的实现。必要的情况下可多属性源可共享一个名字，如使用@PropertySource注解标记加载多个属性文件路径


```java
/**
 * Composite {@link PropertySource} implementation that iterates over a set of
 * {@link PropertySource} instances. Necessary in cases where multiple property sources
 * share the same name, e.g. when multiple values are supplied to {@code @PropertySource}.
 *
 * <p>As of Spring 4.1.2, this class extends {@link EnumerablePropertySource} instead
 * of plain {@link PropertySource}, exposing {@link #getPropertyNames()} based on the
 * accumulated property names from all contained sources (as far as possible).尽可能的从包含的所有源中累计属性名
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @author Phillip Webb
 * @since 3.1.1
 */
public class CompositePropertySource extends EnumerablePropertySource<Object> {

	private final Set<PropertySource<?>> propertySources = new LinkedHashSet<>(); //属性源集合


	/**
	 * Create a new {@code CompositePropertySource}.
	 * @param name the name of the property source
	 */
	public CompositePropertySource(String name) {
		super(name);
	}


	@Override
	@Nullable
	public Object getProperty(String name) { //遍历获取PropertySource集合，获取对应的值对象
		for (PropertySource<?> propertySource : this.propertySources) {
			Object candidate = propertySource.getProperty(name);
			if (candidate != null) {
				return candidate;
			}
		}
		return null;
	}

	@Override
	public boolean containsProperty(String name) {
		for (PropertySource<?> propertySource : this.propertySources) {
			if (propertySource.containsProperty(name)) {
				return true;
			}
		}
		return false;
	}

	@Override
	public String[] getPropertyNames() { //遍历获取所有的属性名
		Set<String> names = new LinkedHashSet<>();
		for (PropertySource<?> propertySource : this.propertySources) {
			if (!(propertySource instanceof EnumerablePropertySource)) {
				throw new IllegalStateException(
						"Failed to enumerate property names due to non-enumerable property source: " + propertySource);
			}
			names.addAll(Arrays.asList(((EnumerablePropertySource<?>) propertySource).getPropertyNames()));
		}
		return StringUtils.toStringArray(names);
	}


	/**
	 * Add the given {@link PropertySource} to the end of the chain. 链表后边添加
	 * @param propertySource the PropertySource to add
	 */
	public void addPropertySource(PropertySource<?> propertySource) {
		this.propertySources.add(propertySource);
	}

	/**
	 * Add the given {@link PropertySource} to the start of the chain. 链表前面添加
	 * @param propertySource the PropertySource to add
	 * @since 4.1
	 */
	public void addFirstPropertySource(PropertySource<?> propertySource) {
		List<PropertySource<?>> existing = new ArrayList<>(this.propertySources);
		this.propertySources.clear();
		this.propertySources.add(propertySource); //添加到首节点
		this.propertySources.addAll(existing);//添加已存在的节点
	}

	/**
	 * Return all property sources that this composite source holds. ：返回此组合源所包含的所有属性源
	 * @since 4.1.1
	 */
	public Collection<PropertySource<?>> getPropertySources() {
		return this.propertySources;
	}

}
```

## ServletConfigPropertySource:Servlet配置属性源

从ServletConfig对象中读取初始化参数

```java
/**
 * {@link PropertySource} that reads init parameters from a {@link ServletConfig} object.
 *
 * @author Chris Beams
 * @since 3.1
 * @see ServletContextPropertySource
 */
public class ServletConfigPropertySource extends EnumerablePropertySource<ServletConfig> {

	public ServletConfigPropertySource(String name, ServletConfig servletConfig) {
		super(name, servletConfig);
	}

	@Override
	public String[] getPropertyNames() {
		return StringUtils.toStringArray(this.source.getInitParameterNames());
	}

	@Override
	@Nullable
	public String getProperty(String name) {
		return this.source.getInitParameter(name);
	}

}

```

## ServletContextPropertySource: ServletContext属性源

从ServletContext对象中读取初始化参数

```java
/**
 * {@link PropertySource} that reads init parameters from a {@link ServletContext} object.
 *
 * @author Chris Beams
 * @since 3.1
 * @see ServletConfigPropertySource
 */
public class ServletContextPropertySource extends EnumerablePropertySource<ServletContext> {

	public ServletContextPropertySource(String name, ServletContext servletContext) {
		super(name, servletContext);
	}

	@Override
	public String[] getPropertyNames() {
		return StringUtils.toStringArray(this.source.getInitParameterNames());
	}

	@Override
	@Nullable
	public String getProperty(String name) {
		return this.source.getInitParameter(name);
	}

}
```

## MapPropertySource：Map属性源

从Map对象中读取key和value的PropertySource实现

```java
/**
 * {@link PropertySource} that reads keys and values from a {@code Map} object.
 *
 * @since 3.1
 * @see PropertiesPropertySource
 */
public class MapPropertySource extends EnumerablePropertySource<Map<String, Object>> {

	public MapPropertySource(String name, Map<String, Object> source) {
		super(name, source);
	}


	@Override
	@Nullable
	public Object getProperty(String name) {
		return this.source.get(name);
	}

	@Override
	public boolean containsProperty(String name) {
		return this.source.containsKey(name);
	}

	@Override
	public String[] getPropertyNames() {
		return StringUtils.toStringArray(this.source.keySet());
	}

}
```

### PropertiesPropertySource:属性文件属性源

从Properties属性文件中提取属性源

```java
/**
 * {@link PropertySource} implementation that extracts 提取 properties from a
 * {@link java.util.Properties} object.
 *
 * <p>Note that because a {@code Properties} object is technically an
 * {@code <Object, Object>} {@link java.util.Hashtable Hashtable}, one may contain
 * non-{@code String} keys or values. This implementation, however is restricted 限制 to 
 * accessing only {@code String}-based keys and values, in the same fashion as 
 * {@link Properties#getProperty} and {@link Properties#setProperty}.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 */
public class PropertiesPropertySource extends MapPropertySource {

	@SuppressWarnings({"unchecked", "rawtypes"})
	public PropertiesPropertySource(String name, Properties source) {
		super(name, (Map) source);
	}

	protected PropertiesPropertySource(String name, Map<String, Object> source) {
		super(name, source);
	}

}
```

### ResourcePropertySource:资源属性源

ResourcePropertySource继承PropertiesPropertySource，从给定的Resource接口中加载一个Properties对象

```java
/**
 * Subclass of {@link PropertiesPropertySource} that loads a {@link Properties} object
 * from a given {@link org.springframework.core.io.Resource} or resource location such as
 * {@code "classpath:/com/myco/foo.properties"} or {@code "file:/path/to/file.xml"}.
 *
 * <p>Both traditional and XML-based properties file formats are supported; however, in
 * order for XML processing to take effect, the underlying {@code Resource}'s
 * {@link org.springframework.core.io.Resource#getFilename() getFilename()} method must
 * return a non-{@code null} value that ends in {@code ".xml"}.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see org.springframework.core.io.Resource
 * @see org.springframework.core.io.support.EncodedResource
 */
public class ResourcePropertySource extends PropertiesPropertySource {

	/** The original resource name, if different from the given name */
	@Nullable
	private final String resourceName; //原始资源名称，不同于给定的name


	/**
	 * Create a PropertySource having the given name based on Properties
	 * loaded from the given encoded resource. 使用编码的资源
	 */
	public ResourcePropertySource(String name, EncodedResource resource) throws IOException {
		super(name, PropertiesLoaderUtils.loadProperties(resource));
		this.resourceName = getNameForResource(resource.getResource()); //获取资源名称
	}

	/**
	 * Create a PropertySource based on Properties loaded from the given resource.
	 * The name of the PropertySource will be generated based on the 属性源名称为资源描述
	 * {@link Resource#getDescription() description} of the given resource.
	 */
	public ResourcePropertySource(EncodedResource resource) throws IOException {
		super(getNameForResource(resource.getResource()), PropertiesLoaderUtils.loadProperties(resource));
		this.resourceName = null;
	}

	/**
	 * Create a PropertySource having the given name based on Properties
	 * loaded from the given encoded resource.
	 */
	public ResourcePropertySource(String name, Resource resource) throws IOException {
		super(name, PropertiesLoaderUtils.loadProperties(new EncodedResource(resource)));
		this.resourceName = getNameForResource(resource);
	}

	/**
	 * Create a PropertySource based on Properties loaded from the given resource.
	 * The name of the PropertySource will be generated based on the
	 * {@link Resource#getDescription() description} of the given resource.
	 */
	public ResourcePropertySource(Resource resource) throws IOException {
		super(getNameForResource(resource), PropertiesLoaderUtils.loadProperties(new EncodedResource(resource)));
		this.resourceName = null;
	}

	/**
	 * Create a PropertySource having the given name based on Properties loaded from
	 * the given resource location  资源文职 and using the given class loader to load the
	 * resource (assuming it is prefixed with {@code classpath:}).
	 */
	public ResourcePropertySource(String name, String location, ClassLoader classLoader) throws IOException {
		this(name, new DefaultResourceLoader(classLoader).getResource(location));
	}

	/**
	 * Create a PropertySource based on Properties loaded from the given resource
	 * location and use the given class loader to load the resource, assuming it is
	 * prefixed with {@code classpath:}. The name of the PropertySource will be
	 * generated based on the {@link Resource#getDescription() description} of the
	 * resource.
	 */
	public ResourcePropertySource(String location, ClassLoader classLoader) throws IOException {
		this(new DefaultResourceLoader(classLoader).getResource(location));
	}

	/**
	 * Create a PropertySource having the given name based on Properties loaded from
	 * the given resource location. The default thread context class loader will be
	 * used to load the resource (assuming the location string is prefixed with
	 * {@code classpath:}.
	 */
	public ResourcePropertySource(String name, String location) throws IOException {
		this(name, new DefaultResourceLoader().getResource(location));
	}

	/**
	 * Create a PropertySource based on Properties loaded from the given resource
	 * location. The name of the PropertySource will be generated based on the
	 * {@link Resource#getDescription() description} of the resource.
	 */
	public ResourcePropertySource(String location) throws IOException {
		this(new DefaultResourceLoader().getResource(location));
	}

	private ResourcePropertySource(String name, @Nullable String resourceName, Map<String, Object> source) {
		super(name, source);
		this.resourceName = resourceName;
	}


	/**
	 返回一个可能经过调整的变体
	 * Return a potentially adapted variant of this {@link ResourcePropertySource},
	 * overriding the previously 预先 given (or derived) name with the specified name.
	 * @since 4.0.4
	 */
	public ResourcePropertySource withName(String name) {
		if (this.name.equals(name)) {
			return this;
		}
		// Store the original resource name if necessary... 存储原始资源名称
		if (this.resourceName != null) {
			if (this.resourceName.equals(name)) {
				return new ResourcePropertySource(this.resourceName, null, this.source);
			}
			else {
				return new ResourcePropertySource(name, this.resourceName, this.source);
			}
		}
		else {
			// Current name is resource name -> preserve it in the extra field...
			return new ResourcePropertySource(name, this.name, this.source);
		}
	}

	/**
	 * Return a potentially adapted variant of this {@link ResourcePropertySource},
	 * overriding the previously given name (if any) with the original resource name
	 * (equivalent to the name generated by the name-less constructor variants).
	 * @since 4.1
	 */
	public ResourcePropertySource withResourceName() {
		if (this.resourceName == null) {
			return this;
		}
		return new ResourcePropertySource(this.resourceName, null, this.source);
	}


	/**
	 * Return the description for the given Resource; if the description is 返回给定资源的描述
	 * empty, return the class name of the resource plus its identity hash code.
	 * @see org.springframework.core.io.Resource#getDescription()
	 */
	private static String getNameForResource(Resource resource) {
		String name = resource.getDescription();
		if (!StringUtils.hasText(name)) {
			name = resource.getClass().getSimpleName() + "@" + System.identityHashCode(resource);
		}
		return name;
	}

}
```

### SystemEnvironmentPropertySource:系统环境属性源

专门用于处理系统环境变量，解析时匹配大写或者下划线

```java
/**
 * Specialization 特殊化 of {@link MapPropertySource} designed 设计 for use with
 * {@linkplain AbstractEnvironment#getSystemEnvironment() system environment variables}.
 * Compensates补偿 for constraints约束 in Bash and other shells that do not allow for variables
 * containing the period character句号 and/or hyphen character 短划线; also allows for uppercase
 * variations on property names for more idiomatic 惯用的 shell use.
 *
 * <p>For example, a call to {@code getProperty("foo.bar")} will attempt to find a value
 * for the original property or any 'equivalent' property, returning the first found:
 * <ul>
 * <li>{@code foo.bar} - the original name</li>
 * <li>{@code foo_bar} - with underscores for periods (if any)</li>
 * <li>{@code FOO.BAR} - original, with upper case</li>
 * <li>{@code FOO_BAR} - with underscores and upper case</li>
 * </ul>
 * Any hyphen variant of the above would work as well, or even mix dot/hyphen variants.
 *
 * <p>The same applies for calls to {@link #containsProperty(String)}, which returns
 * {@code true} if any of the above properties are present, otherwise {@code false}.
 *
 * <p>This feature is particularly useful when specifying active or default profiles as 当知道active或者default profile作为环境变量
 * environment variables. The following is not allowable under Bash:
 *
 * <pre class="code">spring.profiles.active=p1 java -classpath ... MyApp</pre>
 *
 * However, the following syntax is permitted 允许 and is also more conventional:
 *
 * <pre class="code">SPRING_PROFILES_ACTIVE=p1 java -classpath ... MyApp</pre>
 *
 * <p>Enable debug- or trace-level logging for this class (or package) for messages
 * explaining when these 'property name resolutions' occur.
 *
 * <p>This property source is included by default in {@link StandardEnvironment}
 * and all its subclasses.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see StandardEnvironment
 * @see AbstractEnvironment#getSystemEnvironment()
 * @see AbstractEnvironment#ACTIVE_PROFILES_PROPERTY_NAME
 */
public class SystemEnvironmentPropertySource extends MapPropertySource {

	/**
	 * Create a new {@code SystemEnvironmentPropertySource} with the given name and
	 * delegating to the given {@code MapPropertySource}.
	 */
	public SystemEnvironmentPropertySource(String name, Map<String, Object> source) {
		super(name, source);
	}


	/**
	 * Return {@code true} if a property with the given name or any underscore/uppercase variant
	 * thereof exists in this property source.
	 */
	@Override
	public boolean containsProperty(String name) {
		return (getProperty(name) != null);
	}

	/**
	 * This implementation returns {@code true} if a property with the given name or
	 * any underscore/uppercase variant thereof exists in this property source.
	 */
	@Override
	@Nullable
	public Object getProperty(String name) {
		String actualName = resolvePropertyName(name); //解析属性名
		if (logger.isDebugEnabled() && !name.equals(actualName)) {
			logger.debug("PropertySource '" + getName() + "' does not contain property '" + name +
					"', but found equivalent '" + actualName + "'");
		}
		return super.getProperty(actualName);
	}

	/**
	 * Check to see if this property source contains a property with the given name, or
	 * any underscore 下划线 / uppercase 大小写 variation thereof. Return the resolved name if one is
	 * found or otherwise the original name. Never returns {@code null}.
	 */
	protected final String resolvePropertyName(String name) {
		Assert.notNull(name, "Property name must not be null");
		String resolvedName = checkPropertyName(name);
		if (resolvedName != null) {
			return resolvedName;
		}
		String uppercasedName = name.toUpperCase();//全部转换为大写
		if (!name.equals(uppercasedName)) {
			resolvedName = checkPropertyName(uppercasedName);
			if (resolvedName != null) {
				return resolvedName;
			}
		}
		return name;
	}

	@Nullable
	private String checkPropertyName(String name) { 检查属性名
		// Check name as-is
		if (containsKey(name)) { //包含直接返回
			return name;
		}
		// Check name with just dots replaced 
		String noDotName = name.replace('.', '_'); //.全部替换为_
		if (!name.equals(noDotName) && containsKey(noDotName)) {
			return noDotName;
		}
		// Check name with just hyphens replaced
		String noHyphenName = name.replace('-', '_');
		if (!name.equals(noHyphenName) && containsKey(noHyphenName)) {
			return noHyphenName;
		}
		// Check name with dots and hyphens replaced
		String noDotNoHyphenName = noDotName.replace('-', '_');
		if (!noDotName.equals(noDotNoHyphenName) && containsKey(noDotNoHyphenName)) {
			return noDotNoHyphenName;
		}
		// Give up
		return null;
	}

	private boolean containsKey(String name) { //是否包含
		return (isSecurityManagerPresent() ? this.source.keySet().contains(name) : this.source.containsKey(name));
	}

	protected boolean isSecurityManagerPresent() {
		return (System.getSecurityManager() != null);
	}

}

```
