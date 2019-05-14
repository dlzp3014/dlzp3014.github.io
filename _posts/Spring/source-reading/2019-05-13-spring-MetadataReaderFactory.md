---
layout: post
title:  "Spring源码-MetadataReaderFactory元数据读取工厂"
date:   2019-05-13 22:18:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

MetadataReaderFactory为MetadataReader实例的工厂接口，即主要用于创建MetadataReader元数据读取器，从而读取class文件的相关元数据信息，MetadataReader请参考：[Spring源码-MetadataReader元数据读取器接口](/2019/05/14/spring-MetadataReader/)。Spring提供的实现有SimpleMetadataReaderFactory、CachingMetadataReaderFactory、ConcurrentReferenceCachingMetadataReaderFactory。类结构如下：
![](/img/post.img/spring/MetadataReaderFactory.png)






```java
/**
 * Factory interface for {@link MetadataReader} instances.
 * Allows for caching a MetadataReader per original resource. 允许缓存每个原始资源的元数据阅读器
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see SimpleMetadataReaderFactory 简单元数据读取器工厂
 * @see CachingMetadataReaderFactory 缓存元数据读取器工厂
 */
public interface MetadataReaderFactory {

	/**
	 * Obtain a MetadataReader for the given class name. 从给定的类名中获取MetadataReader
	 * @param className the class name (to be resolved to a ".class" file) 类名，用于解析class文件
	 * @return a holder for the ClassReader instance (never {@code null}) 类阅读器实例的持有者
	 * @throws IOException in case of I/O failure
	 */
	MetadataReader getMetadataReader(String className) throws IOException;

	/**
	 * Obtain a MetadataReader for the given resource. 从给定资源中获取MetadataReader
	 * @param resource the resource (pointing to a ".class" file) 资源，指向class文件
	 * @return a holder for the ClassReader instance (never {@code null})
	 * @throws IOException in case of I/O failure
	 */
	MetadataReader getMetadataReader(Resource resource) throws IOException;

}

```

## SimpleMetadataReaderFactory：简单元数据读取器工厂

SimpleMetadataReaderFactory中默认使用DefaultResourceLoader去加载class文件，然后使用SimpleMetadataReader构造返回MetadataReader


### SimpleMetadataReaderFactory：类实现过程

```java

/**
 * Simple implementation of the {@link MetadataReaderFactory} interface,
 * creating a new ASM {@link org.springframework.asm.ClassReader} for every request. 每次请求都会创建一个新的ASM ClassReader，所以当前类时线程安全的
 *
 * @author Juergen Hoeller
 * @since 2.5
 */
public class SimpleMetadataReaderFactory implements MetadataReaderFactory {

	private final ResourceLoader resourceLoader; //资源加载器


	/**
	 * Create a new SimpleMetadataReaderFactory for the default class loader.创建SimpleMetadataReaderFactory使用默认的类加载器
	 */
	public SimpleMetadataReaderFactory() {
		this.resourceLoader = new DefaultResourceLoader();
	}

	/**
	 * Create a new SimpleMetadataReaderFactory for the given resource loader. 给定资源加载器
	 * @param resourceLoader the Spring ResourceLoader to use 使用的Spring ResourceLoader
	 * (also determines the ClassLoader to use)
	 */
	public SimpleMetadataReaderFactory(@Nullable ResourceLoader resourceLoader) {
		this.resourceLoader = (resourceLoader != null ? resourceLoader : new DefaultResourceLoader());
	}

	/**
	 * Create a new SimpleMetadataReaderFactory for the given class loader. 给定类加载器
	 * @param classLoader the ClassLoader to use
	 */
	public SimpleMetadataReaderFactory(@Nullable ClassLoader classLoader) {
		this.resourceLoader =
				(classLoader != null ? new DefaultResourceLoader(classLoader) : new DefaultResourceLoader());
	}


	/**
	 * Return the ResourceLoader that this MetadataReaderFactory has been
	 * constructed with. 
	 */
	public final ResourceLoader getResourceLoader() {
		return this.resourceLoader;
	}


	@Override
	public MetadataReader getMetadataReader(String className) throws IOException {
		try {
			String resourcePath = ResourceLoader.CLASSPATH_URL_PREFIX + //classpath:
					ClassUtils.convertClassNameToResourcePath(className)  //将类名转换为向对于目录
					+ ClassUtils.CLASS_FILE_SUFFIX;//.class
			Resource resource = this.resourceLoader.getResource(resourcePath); //使用资源加载器加载类
			return getMetadataReader(resource); //通过Resource获取MetadataReader对象
		}
		catch (FileNotFoundException ex) {
			// Maybe an inner class name using the dot name syntax? Need to use the dollar syntax($语法) here... 内部类处理
			// ClassUtils.forName has an equivalent check for resolution into Class references later on. 稍后解析为类引用
			int lastDotIndex = className.lastIndexOf('.'); 
			if (lastDotIndex != -1) { //类名中使用包含.
				String innerClassName = //内部类名outClassName+$+innerClassName
						className.substring(0, lastDotIndex) + '$' + className.substring(lastDotIndex + 1);
				String innerClassResourcePath = ResourceLoader.CLASSPATH_URL_PREFIX +
						ClassUtils.convertClassNameToResourcePath(innerClassName) + ClassUtils.CLASS_FILE_SUFFIX;
				Resource innerClassResource = this.resourceLoader.getResource(innerClassResourcePath);
				if (innerClassResource.exists()) { //资源文件存在
					return getMetadataReader(innerClassResource);
				}
			}
			throw ex;
		}
	}

	@Override
	public MetadataReader getMetadataReader(Resource resource) throws IOException {
		return new SimpleMetadataReader(resource, this.resourceLoader.getClassLoader());
	}

}


```
### SimpleMetadataReaderFactory：使用实例(获取类中元数据的信息)

```java
@Service
public class CoreMain {

    @Test
    @Bean(value = "testBean")
    public void test() throws IOException {
        MetadataReaderFactory metadataReaderFactory = new SimpleMetadataReaderFactory(); 
        //元数据赌气器
        MetadataReader metadataReader = metadataReaderFactory.getMetadataReader(CoreMain.class.getName());
        //获取类上面的注解
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();

        Set<String> annotationTypes = annotationMetadata.getAnnotationTypes();

        Assert.assertEquals(Collections.singleton("org.springframework.stereotype.Service"), annotationTypes);

        //获取指定注解上的方法
        Set<MethodMetadata> annotatedMethods = annotationMetadata.getAnnotatedMethods(Bean.class.getName());
        annotatedMethods.forEach(methodMetadata -> {
            String methodName = methodMetadata.getMethodName(); //方法名
            Assert.assertEquals("test", methodName);

            MultiValueMap<String, Object> beanAnnotationAttributes = methodMetadata.getAllAnnotationAttributes(Bean.class.getName());
            
            String beanValue = ((String[]) beanAnnotationAttributes.getFirst("value"))[0]; //注解属性中的值
            Assert.assertEquals("testBean", beanValue);
            
        });

    }
}

```

## CachingMetadataReaderFactory：MetadataReaderFactory接口的缓存实现

CachingMetadataReaderFactory缓存每一个Resource的MetadataReader对象


```java

/**
 * Caching implementation of the {@link MetadataReaderFactory} interface,
 * caching a {@link MetadataReader} instance per Spring {@link Resource} handle
 * (i.e. per ".class" file).
 *
 * @author Juergen Hoeller
 * @author Costin Leau
 * @since 2.5
 */
public class CachingMetadataReaderFactory extends SimpleMetadataReaderFactory {

	/** Default maximum number of entries for a local MetadataReader cache: 256 */ 默认最大缓存大小
	public static final int DEFAULT_CACHE_LIMIT = 256;

	/** MetadataReader cache: either local or shared at the ResourceLoader level */ 在ResourceLoader级别上本地或共享资源
	@Nullable
	private Map<Resource, MetadataReader> metadataReaderCache;


	/**
	 * Create a new CachingMetadataReaderFactory for the default class loader,
	 * using a local resource cache. 使用本地资源缓存
	 */
	public CachingMetadataReaderFactory() {
		super();
		setCacheLimit(DEFAULT_CACHE_LIMIT);
	}

	/**
	 * Create a new CachingMetadataReaderFactory for the given {@link ClassLoader},
	 * using a local resource cache.
	 * @param classLoader the ClassLoader to use
	 */
	public CachingMetadataReaderFactory(@Nullable ClassLoader classLoader) {
		super(classLoader);
		setCacheLimit(DEFAULT_CACHE_LIMIT);
	}

	/**
	 * Create a new CachingMetadataReaderFactory for the given {@link ResourceLoader},
	 * using a shared resource cache if supported or a local resource cache otherwise. 如果支持使用共享的资源缓存，否则使用本地资源缓存
	 * @param resourceLoader the Spring ResourceLoader to use
	 * (also determines the ClassLoader to use)
	 * @see DefaultResourceLoader#getResourceCache
	 */
	public CachingMetadataReaderFactory(@Nullable ResourceLoader resourceLoader) {
		super(resourceLoader);
		if (resourceLoader instanceof DefaultResourceLoader) {
			this.metadataReaderCache =
					((DefaultResourceLoader) resourceLoader).getResourceCache(MetadataReader.class);
		}
		else {
			setCacheLimit(DEFAULT_CACHE_LIMIT);
		}
	}


	/**
	 * Specify the maximum number of entries for the MetadataReader cache. 指定一个最大用于缓存MetadataReader的值
	 * <p>Default is 256 for a local cache, whereas a shared cache is 默认本地缓存大小为256，然而共享缓存不受显式
	 * typically unbounded. This method enforces a local resource cache, 这个方法强制执行本地缓存
	 * even if the {@link ResourceLoader} supports a shared resource cache. 即使ResourceLoader支持共享资源缓存
	 */
	public void setCacheLimit(int cacheLimit) {
		if (cacheLimit <= 0) {
			this.metadataReaderCache = null;
		}
		else if (this.metadataReaderCache instanceof LocalResourceCache) {
			((LocalResourceCache) this.metadataReaderCache).setCacheLimit(cacheLimit);
		}
		else {
			this.metadataReaderCache = new LocalResourceCache(cacheLimit);
		}
	}

	/**
	 * Return the maximum number of entries for the MetadataReader cache.
	 */
	public int getCacheLimit() {
		if (this.metadataReaderCache instanceof LocalResourceCache) {
			return ((LocalResourceCache) this.metadataReaderCache).getCacheLimit();
		}
		else {
			return (this.metadataReaderCache != null ? Integer.MAX_VALUE : 0);
		}
	}


	@Override
	public MetadataReader getMetadataReader(Resource resource) throws IOException {
		if (this.metadataReaderCache instanceof ConcurrentMap) { //并发容器
			// No synchronization necessary... 没有必要同步
			MetadataReader metadataReader = this.metadataReaderCache.get(resource);  //从缓存中读取,为null是获取，MetadataReader
			if (metadataReader == null) {
				metadataReader = super.getMetadataReader(resource); 
				this.metadataReaderCache.put(resource, metadataReader); //加入缓存
			}
			return metadataReader;
		}
		else if (this.metadataReaderCache != null) {
			synchronized (this.metadataReaderCache) { //加锁处理
				MetadataReader metadataReader = this.metadataReaderCache.get(resource);
				if (metadataReader == null) {
					metadataReader = super.getMetadataReader(resource);
					this.metadataReaderCache.put(resource, metadataReader);
				}
				return metadataReader;
			}
		}
		else {
			return super.getMetadataReader(resource);
		}
	}

	/**
	 * Clear the local MetadataReader cache, if any, removing all cached class metadata. 移除说有缓存类元数据
	 */
	public void clearCache() {
		if (this.metadataReaderCache instanceof LocalResourceCache) {
			synchronized (this.metadataReaderCache) {
				this.metadataReaderCache.clear();
			}
		}
		else if (this.metadataReaderCache != null) {
			// Shared resource cache -> reset to local cache.
			setCacheLimit(DEFAULT_CACHE_LIMIT); //重置到本地缓存
		}
	}


	@SuppressWarnings("serial")
	private static class LocalResourceCache extends LinkedHashMap<Resource, MetadataReader> {

		private volatile int cacheLimit; //缓存大小

		public LocalResourceCache(int cacheLimit) {
			super(cacheLimit, 0.75f, true);
			this.cacheLimit = cacheLimit;
		}

		public void setCacheLimit(int cacheLimit) {
			this.cacheLimit = cacheLimit;
		}

		public int getCacheLimit() {
			return this.cacheLimit;
		}

		@Override
		protected boolean removeEldestEntry(Map.Entry<Resource, MetadataReader> eldest) {
			return size() > this.cacheLimit;
		}
	}

}


```

### ConcurrentReferenceCachingMetadataReaderFactory：并发引用缓存元数据读取器工厂

```java
/**
 * Caching implementation of the {@link MetadataReaderFactory} interface backed by a 依靠
 * {@link ConcurrentReferenceHashMap}, caching {@link MetadataReader} per Spring
 * {@link Resource} handle (i.e. per ".class" file).
 *
 * @author Phillip Webb
 * @since 1.4.0
 * @see CachingMetadataReaderFactory
 */
public class ConcurrentReferenceCachingMetadataReaderFactory
		extends SimpleMetadataReaderFactory {

	private final Map<Resource, MetadataReader> cache = new ConcurrentReferenceHashMap<>(); //并发引入Map

	/**
	 * Create a new {@link ConcurrentReferenceCachingMetadataReaderFactory} instance for
	 * the default class loader.
	 */
	public ConcurrentReferenceCachingMetadataReaderFactory() {
	}

	/**
	 * Create a new {@link ConcurrentReferenceCachingMetadataReaderFactory} instance for
	 * the given resource loader.
	 * @param resourceLoader the Spring ResourceLoader to use (also determines the
	 * ClassLoader to use)
	 */
	public ConcurrentReferenceCachingMetadataReaderFactory(
			ResourceLoader resourceLoader) {
		super(resourceLoader);
	}

	/**
	 * Create a new {@link ConcurrentReferenceCachingMetadataReaderFactory} instance for
	 * the given class loader.
	 * @param classLoader the ClassLoader to use
	 */
	public ConcurrentReferenceCachingMetadataReaderFactory(ClassLoader classLoader) {
		super(classLoader);
	}

	@Override
	public MetadataReader getMetadataReader(Resource resource) throws IOException {
		MetadataReader metadataReader = this.cache.get(resource);
		if (metadataReader == null) {
			metadataReader = createMetadataReader(resource);
			this.cache.put(resource, metadataReader);
		}
		return metadataReader;
	}

	/**
	 * Create the meta-data reader. 创建元数据读取器
	 * @param resource the source resource.
	 * @return the meta-data reader
	 * @throws IOException on error
	 */
	protected MetadataReader createMetadataReader(Resource resource) throws IOException {
		return super.getMetadataReader(resource);
	}

	/**
	 * Clear the entire MetadataReader cache, removing all cached class metadata.
	 */
	public void clearCache() {
		this.cache.clear();
	}

}

```
