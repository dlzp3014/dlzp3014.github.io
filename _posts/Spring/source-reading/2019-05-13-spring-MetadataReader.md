---
layout: post
title:  "Spring源码-MetadataReader元数据读取器接口"
date:   2019-05-13 22:01:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}


MetadataReader是一个简单的门面接口，通过ASM ClassReader类读取器来访问类的元数据，Spring提供了简单的实现：SimpleMetadataReader




## MetadataReader:接口定义

```java
/**
 * Simple facade for accessing class metadata,
 * as(由) read by an ASM {@link org.springframework.asm.ClassReader}.
 * @since 2.5
 */
public interface MetadataReader {

	/**
	 * Return the resource reference for the class file.返回类文件的资源引用
	 */
	Resource getResource();

	/**
	 * Read basic class metadata for the underlying class. 读取基础类的基本类元数据
	 */
	ClassMetadata getClassMetadata();

	/**
	 * Read full annotation metadata for the underlying class, 读取底层类的完整注释元数据，
	 * including metadata for annotated methods. 包括带注释方法的元数据
	 */
	AnnotationMetadata getAnnotationMetadata();

}

```

## SimpleMetadataReader:MetadataReader接口实现

SimpleMetadataReader为包级可见，可以通过SimpleMetadataReaderFactory#getMetadataReader(Resource resource) 获得，具体请参考:[Spring源码-MetadataReaderFactory元数据读取工厂类](/2019/05/14/spring-MetadataReaderFactory/)。

SimpleMetadataReader内部主要使用ClassReader加载类文件，然后通过AnnotationMetadataReadingVisitor注解元数据读取访问者来获取类元素数据的相关信息

```java
/**
 * {@link MetadataReader} implementation based on an ASM 
 * {@link org.springframework.asm.ClassReader}.
 *
 * <p>Package-visible in order to allow for repackaging the ASM library 为了重新包装ASM库而包级乐见，
 * without effect on users of the {@code core.type} package. 不影响core.type的包的使用
 *
 * @since 2.5
 */
final class SimpleMetadataReader implements MetadataReader {

	private final Resource resource; //类文件资源引用

	private final ClassMetadata classMetadata; //类元素
 
	private final AnnotationMetadata annotationMetadata; //注解元素

	//通过resource及classLoader创建SimpleMetadataReader实例
	SimpleMetadataReader(Resource resource, @Nullable ClassLoader classLoader) throws IOException {
		InputStream is = new BufferedInputStream(resource.getInputStream()); //缓冲区输入流
		ClassReader classReader;
		try {
			classReader = new ClassReader(is); //asm 类加载器
		}
		catch (IllegalArgumentException ex) {
			throw new NestedIOException("ASM ClassReader failed to parse class file - " +
					"probably due to a new Java class file version that isn't supported yet: " + resource, ex);
		}
		finally {
			is.close();
		}

		AnnotationMetadataReadingVisitor visitor = new AnnotationMetadataReadingVisitor(classLoader); //创建注解元数据读取访问者对象
		classReader.accept(visitor, ClassReader.SKIP_DEBUG); //忽略debug模式

		this.annotationMetadata = visitor;
		// (since AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor)
		this.classMetadata = visitor;
		this.resource = resource;
	}


	@Override
	public Resource getResource() {
		return this.resource;
	}

	@Override
	public ClassMetadata getClassMetadata() {
		return this.classMetadata;
	}

	@Override
	public AnnotationMetadata getAnnotationMetadata() {
		return this.annotationMetadata;
	}

}

```





