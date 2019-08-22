---
layout: post
title:  "Spring源码-SpringFactoriesLoader:SPI加载器"
date:   2019-08-22 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

SpringFactoriesLoader用于加载和初始化来自`META-INF/spring.factories`配置文件中给定的类，其中配置文件spring.factories可能存在于多个Jar文件中。spring.factories文件必须是Properties格式化：key为全限定接口或者抽象类，value为逗号`,`分割的实现类列表名，如下

```java

example.IService=example.ServiceImplA,example.ServiceImplB
```

`SpringFactoriesLoader为Spring提供的SPI服务提供者接口加载、初始化的实现。在SpringBoot的SpringApplication类中大量使用，用于加载自动化配置类的实现`





```java
public abstract class SpringFactoriesLoader {

	/**
	 * The location to look for factories.
	 * <p>Can be present in multiple JAR files.
	 */
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";


	private static final Log logger = LogFactory.getLog(SpringFactoriesLoader.class);

	//key 为当前线程的类加载器，value为接口或者抽象类即多个实现的多值映射
	private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();


	/**
	 * Load and instantiate the factory implementations of the given type from
	 * {@value #FACTORIES_RESOURCE_LOCATION}, using the given class loader. 使用给定的类加载器，加载和初始化来自spring.factories中给定的类型
	 * <p>The returned factories are sorted through {@link AnnotationAwareOrderComparator}. 排查
	 * <p>If a custom instantiation strategy is required, use {@link #loadFactoryNames} 如果需要自定义实例化策略，使用loadFactoryNames获取所有已注册的工厂名称
	 * to obtain all registered factory names.
	 * @param factoryClass the interface or abstract class representing 代表 the factory 
	 * @param classLoader the ClassLoader to use for loading (can be {@code null} to use the default)
	 * @throws IllegalArgumentException if any factory implementation class cannot
	 * be loaded or if an error occurs while instantiating any factory
	 * @see #loadFactoryNames
	 */
	public static <T> List<T> loadFactories(Class<T> factoryClass, @Nullable ClassLoader classLoader) {
		Assert.notNull(factoryClass, "'factoryClass' must not be null");
		ClassLoader classLoaderToUse = classLoader;
		if (classLoaderToUse == null) {
			classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
		}
		List<String> factoryNames = loadFactoryNames(factoryClass, classLoaderToUse); //获取工厂类对应的实现列表
		if (logger.isTraceEnabled()) {
			logger.trace("Loaded [" + factoryClass.getName() + "] names: " + factoryNames);
		}
		List<T> result = new ArrayList<>(factoryNames.size());
		for (String factoryName : factoryNames) {
			result.add(instantiateFactory(factoryName, factoryClass, classLoaderToUse)); //初始化
		}
		AnnotationAwareOrderComparator.sort(result); //排序
		return result;
	}

	/**
	 * Load the fully qualified class names of factory implementations of the 加载全限定类实现的类名
	 * given type from {@value #FACTORIES_RESOURCE_LOCATION}, using the given
	 * class loader.
	 * @param factoryClass the interface or abstract class representing the factory
	 * @param classLoader the ClassLoader to use for loading resources; can be
	 * {@code null} to use the default
	 * @throws IllegalArgumentException if an error occurs while loading factory names
	 * @see #loadFactories
	 */
	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}
	//获取类加载器对应的接口或者抽象类对应的实现类列表
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader); //类加载器对应的值
		if (result != null) {
			return result;
		}

		try {
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) : //使用当前类加载器获取所有jar中的spring.factories文件
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url); //url资源
				Properties properties = PropertiesLoaderUtils.loadProperties(resource); //加载为Properties属性
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) { //,分割
						result.add(factoryClassName, factoryName.trim()); //接口名 1->n 实现
					}
				}
			}
			cache.put(classLoader, result); //当前缓存
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}

	@SuppressWarnings("unchecked") //实例化具体实现类
	private static <T> T instantiateFactory(String instanceClassName, Class<T> factoryClass, ClassLoader classLoader) {
		try {
			Class<?> instanceClass = ClassUtils.forName(instanceClassName, classLoader);//实例化，使用默认的构造函数
			if (!factoryClass.isAssignableFrom(instanceClass)) {
				throw new IllegalArgumentException(
						"Class [" + instanceClassName + "] is not assignable to [" + factoryClass.getName() + "]");
			}
			return (T) ReflectionUtils.accessibleConstructor(instanceClass).newInstance(); //使用构造函数实例化对象，并进行强制转换
		}
		catch (Throwable ex) {
			throw new IllegalArgumentException("Unable to instantiate factory class: " + factoryClass.getName(), ex);
		}
	}

}
```


