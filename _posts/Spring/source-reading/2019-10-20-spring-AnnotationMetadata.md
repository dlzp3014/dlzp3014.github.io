---
layout: post
title:  "Spring源码-AnnotationMetadata注解元数据接口"
date:   2019-10-19 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

AnnotationMetadata接口继承了ClassMetadata类元数据接口(具体请查看：[Spring源码-ClassMetadata类元数据接口](/2019/10/19/spring-ClassMetadata/))和AnnotatedTypeMetadata(具体请查看：[Spring源码-AnnotatedTypeMetadata注解类型元数据接口](/2019/10/19/spring-AnnotatedTypeMetadata/))注解类型元数据接口，定义了访问特定类型的注解。Spring提供的默认实现有StandardAnnotationMetadata(使用反射内省给定的Class)和AnnotationMetadataReadingVisitor(ASM class visitor)，类结果如下：

![](/img/post.img/spring/AnnotationMetadata.png)






## AnnotationMetadata：注解元数据接口

AnnotationMetadata接口定义了抽象访问指定类的注解，而不需要加载类。源代码如下：

```java
/**
 * Interface that defines abstract access to the annotations of a specific
 * class, in a form that does not require that class to be loaded yet.
 *
 * @since 2.5
 * @see StandardAnnotationMetadata
 * @see org.springframework.core.type.classreading.MetadataReader#getAnnotationMetadata()
 * @see AnnotatedTypeMetadata
 */
public interface AnnotationMetadata extends ClassMetadata, AnnotatedTypeMetadata {

	/** 获取出现在类上所有注解类型的全部全限类名
	 * Get the fully qualified class names of all annotation types that
	 * are <em>present</em> on the underlying class.
	 * @return the annotation type names
	 */
	Set<String> getAnnotationTypes();

	/**
	 * Get the fully qualified class names of all meta-annotation types元注解类型 that
	 * are <em>present</em> on the given annotation type on the underlying class.
	 * @param annotationName the fully qualified class name of the meta-annotation
	 * type to look for 要查找的元注解类型的全限类名
	 * @return the meta-annotation type names, or an empty set if none found
	 */
	Set<String> getMetaAnnotationTypes(String annotationName);

	/**确认给定的注解类型是否存在于基础类中
	 * Determine whether an annotation of the given type is <em>present</em> on
	 * the underlying class.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for
	 * @return {@code true} if a matching annotation is present
	 */
	boolean hasAnnotation(String annotationName);

	/**确定基础类是否有一个自身的注释使用给定类型的元注释进行注释：组合注解
	 * Determine whether the underlying class has an annotation that is itself
	 * annotated with the meta-annotation of the given type.
	 * @param metaAnnotationName the fully qualified class name of the
	 * meta-annotation type to look for
	 * @return {@code true} if a matching meta-annotation is present
	 */
	boolean hasMetaAnnotation(String metaAnnotationName);

	/**确认基础类是否有给定注解类型注释或者元注释的方法：即类中的方法有指定的注释
	 * Determine whether the underlying class has any methods that are
	 * annotated (or meta-annotated) with the given annotation type.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for
	 */
	boolean hasAnnotatedMethods(String annotationName);

	/**检索所有方法的方法元数据，使用给定的注解类型注释
	 * Retrieve the method metadata for all methods that are annotated
	 * (or meta-annotated) with the given annotation type.
	 * <p>For any returned method, {@link MethodMetadata#isAnnotated} will 
	 * return {@code true} for the given annotation type.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for 方法的方法元数据集合：匹配一个注释
	 * @return a set of {@link MethodMetadata} for methods that have a matching
	 * annotation. The return value will be an empty set if no methods match
	 * the annotation type.
	 */
	Set<MethodMetadata> getAnnotatedMethods(String annotationName);

}
```

## StandardAnnotationMetadata：标准注解元数据

StandardAnnotationMetadata继承了StandardClassMetadata标准类元数据，具体请查看：[Spring源码-ClassMetadata类元数据接口](/2019/10/19/spring-ClassMetadata/)，并实现了AnnotationMetadata接口，`使用标准反射内省一个给定的Class`


```java
public class StandardAnnotationMetadata extends StandardClassMetadata implements AnnotationMetadata {

	private final Annotation[] annotations; //类中的所有注释

	private final boolean nestedAnnotationsAsMap; //类注释是否作为AnnotationAttribute对象


	/**
	 * Create a new {@code StandardAnnotationMetadata} wrapper for the given Class.
	 * @param introspectedClass the Class to introspect
	 * @see #StandardAnnotationMetadata(Class, boolean)
	 */
	public StandardAnnotationMetadata(Class<?> introspectedClass) {
		this(introspectedClass, false);
	}

	/**
	 * Create a new {@link StandardAnnotationMetadata} wrapper for the given Class,
	 * providing the option to return any nested annotations 嵌套注释的选项 or annotation arrays in the
	 * form of {@link org.springframework.core.annotation.AnnotationAttributes} instead
	 * of actual {@link Annotation} instances. AnnotationAttributes代替真实的Annotation实例
	 * @param introspectedClass the Class to introspect
	 * @param nestedAnnotationsAsMap return nested annotations 返回嵌套注释 and annotation arrays as
	 * {@link org.springframework.core.annotation.AnnotationAttributes} for compatibility 兼容性
	 * with ASM-based {@link AnnotationMetadata} implementations
	 * @since 3.1.1
	 */
	public StandardAnnotationMetadata(Class<?> introspectedClass, boolean nestedAnnotationsAsMap) {
		super(introspectedClass); //StandardClassMetadata 构造函数
		this.annotations = introspectedClass.getAnnotations(); //直接获取类的所有注解
		this.nestedAnnotationsAsMap = nestedAnnotationsAsMap; 
	}


	@Override
	public Set<String> getAnnotationTypes() {
		Set<String> types = new LinkedHashSet<>();
		for (Annotation ann : this.annotations) {
			types.add(ann.annotationType().getName()); //注释类名
		}
		return types;
	}

	@Override //注解元数据类型
	public Set<String> getMetaAnnotationTypes(String annotationName) {
		return (this.annotations.length > 0 ?
				AnnotatedElementUtils.getMetaAnnotationTypes(getIntrospectedClass(), annotationName) :
				Collections.emptySet());
	}

	@Override
	public boolean hasAnnotation(String annotationName) {
		for (Annotation ann : this.annotations) {
			if (ann.annotationType().getName().equals(annotationName)) {
				return true;
			}
		}
		return false;
	}

	@Override
	public boolean hasMetaAnnotation(String annotationName) {
		return (this.annotations.length > 0 &&
				AnnotatedElementUtils.hasMetaAnnotationTypes(getIntrospectedClass(), annotationName));
	}

	@Override
	public boolean isAnnotated(String annotationName) {
		return (this.annotations.length > 0 &&
				AnnotatedElementUtils.isAnnotated(getIntrospectedClass(), annotationName));
	}

	@Override
	public Map<String, Object> getAnnotationAttributes(String annotationName) {
		return getAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		return (this.annotations.length > 0 ? AnnotatedElementUtils.getMergedAnnotationAttributes(
				getIntrospectedClass(), annotationName, classValuesAsString, this.nestedAnnotationsAsMap) : null); 合并，重写注解属性
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName) {
		return getAllAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		return (this.annotations.length > 0 ? AnnotatedElementUtils.getAllAnnotationAttributes(
				getIntrospectedClass(), annotationName, classValuesAsString, this.nestedAnnotationsAsMap) : null); 获取所有注释属性
	}

	@Override 方法上注释
	public boolean hasAnnotatedMethods(String annotationName) {
		try {
			Method[] methods = getIntrospectedClass().getDeclaredMethods();
			for (Method method : methods) {
				桥接方法、由注释且为指定的注释
				if (!method.isBridge() && method.getAnnotations().length > 0 &&
						AnnotatedElementUtils.isAnnotated(method, annotationName)) {
					return true;
				}
			}
			return false;
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Failed to introspect annotated methods on " + getIntrospectedClass(), ex);
		}
	}

	@Override 注释方法元数据
	public Set<MethodMetadata> getAnnotatedMethods(String annotationName) {
		try {
			Method[] methods = getIntrospectedClass().getDeclaredMethods();
			Set<MethodMetadata> annotatedMethods = new LinkedHashSet<>(4);
			for (Method method : methods) {
				if (!method.isBridge() && method.getAnnotations().length > 0 &&
						AnnotatedElementUtils.isAnnotated(method, annotationName)) {
					//标准方法元数据
					annotatedMethods.add(new StandardMethodMetadata(method, this.nestedAnnotationsAsMap));
				}
			}
			return annotatedMethods;
		}
		catch (Throwable ex) {
			throw new IllegalStateException("Failed to introspect annotated methods on " + getIntrospectedClass(), ex);
		}
	}

}

```


## AnnotationMetadataReadingVisitor：注解元数据读取访问器

AnnotationMetadataReadingVisitor继承了ClassMetadataReadingVisitor标准类元数据，具体请查看：[Spring源码-ClassMetadata类元数据接口#ClassMetadataReadingVisitor:类元数据读取访问器](/2019/10/19/spring-ClassMetadata/#classmetadatareadingvisitor类元数据读取访问器)，并实现了AnnotationMetadata接口，`使用ASM 类访问器查找类上定义的注释`

```java
/**
 * ASM class visitor which looks for the class name类名 and implemented types实现类型 as
 * well as for the annotations defined on the class 以及类上定义的注释, exposing them through
 * the {@link org.springframework.core.type.AnnotationMetadata} interface. 通过AnnotationMetadata暴露
 *
 * @since 2.5
 */
public class AnnotationMetadataReadingVisitor extends ClassMetadataReadingVisitor implements AnnotationMetadata {

	@Nullable
	protected final ClassLoader classLoader; 类加载器

	protected final Set<String> annotationSet = new LinkedHashSet<>(4); 注解集合

	protected final Map<String, Set<String>> metaAnnotationMap = new LinkedHashMap<>(4); 元数据注解映射为map,key为全限定注解名，值为源注解

	/** 声明一个链表多值Map代替MultiValueMap，以确保保留条目的层次顺序
	 * Declared as a {@link LinkedMultiValueMap} instead of a {@link MultiValueMap}
	 * to ensure that the hierarchical 分层 ordering of the entries is preserved.
	 * @see AnnotationReadingVisitorUtils#getMergedAnnotationAttributes
	 */
	protected final LinkedMultiValueMap<String, AnnotationAttributes> attributesMap = new LinkedMultiValueMap<>(4);

	protected final Set<MethodMetadata> methodMetadataSet = new LinkedHashSet<>(4); 方法元数据


	public AnnotationMetadataReadingVisitor(@Nullable ClassLoader classLoader) {
		this.classLoader = classLoader;
	}


	@Override 方法访问器
	public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
		忽略桥接方法
		// Skip bridge methods - we're only interested感兴趣 in original annotation-defining user methods.
		// On JDK 8, we'd otherwise run into double detection检测 of the same annotated method...
		if ((access & Opcodes.ACC_BRIDGE) != 0) {
			return super.visitMethod(access, name, desc, signature, exceptions);
		}
		return new MethodMetadataReadingVisitor(name, access, getClassName(),
				Type.getReturnType(desc).getClassName(), this.classLoader, this.methodMetadataSet);
	}

	@Override 注解方法器
	public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
		String className = Type.getType(desc).getClassName();
		this.annotationSet.add(className); 
		return new AnnotationAttributesReadingVisitor(
				className, this.attributesMap, this.metaAnnotationMap, this.classLoader);
	}

	注解
	@Override
	public Set<String> getAnnotationTypes() {
		return this.annotationSet;
	}

	元注解类型
	@Override
	public Set<String> getMetaAnnotationTypes(String annotationName) {
		Set<String> metaAnnotationTypes = this.metaAnnotationMap.get(annotationName);
		return (metaAnnotationTypes != null ? metaAnnotationTypes : Collections.emptySet());
	}

	@Override
	public boolean hasAnnotation(String annotationName) {
		return this.annotationSet.contains(annotationName);
	}

	@Override
	public boolean hasMetaAnnotation(String metaAnnotationType) {
		Collection<Set<String>> allMetaTypes = this.metaAnnotationMap.values();
		for (Set<String> metaTypes : allMetaTypes) {
			if (metaTypes.contains(metaAnnotationType)) {
				return true;
			}
		}
		return false;
	}

	@Override
	public boolean isAnnotated(String annotationName) {
		return (!AnnotationUtils.isInJavaLangAnnotationPackage(annotationName) &&
				this.attributesMap.containsKey(annotationName));
	}

	注解属性
	@Override 
	@Nullable
	public AnnotationAttributes getAnnotationAttributes(String annotationName) {
		return getAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public AnnotationAttributes getAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		AnnotationAttributes raw = AnnotationReadingVisitorUtils.getMergedAnnotationAttributes(
				this.attributesMap, this.metaAnnotationMap, annotationName);
		if (raw == null) {
			return null;
		}
		return AnnotationReadingVisitorUtils.convertClassValues(
				"class '" + getClassName() + "'", this.classLoader, raw, classValuesAsString);
	}

	所有注解属性
	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName) {
		return getAllAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		MultiValueMap<String, Object> allAttributes = new LinkedMultiValueMap<>();
		List<AnnotationAttributes> attributes = this.attributesMap.get(annotationName);
		if (attributes == null) {
			return null;
		}
		for (AnnotationAttributes raw : attributes) {
			for (Map.Entry<String, Object> entry : AnnotationReadingVisitorUtils.convertClassValues(
					"class '" + getClassName() + "'", this.classLoader, raw, classValuesAsString).entrySet()) {
				allAttributes.add(entry.getKey(), entry.getValue());
			}
		}
		return allAttributes;
	}

	@Override
	public boolean hasAnnotatedMethods(String annotationName) {
		for (MethodMetadata methodMetadata : this.methodMetadataSet) {
			if (methodMetadata.isAnnotated(annotationName)) {
				return true;
			}
		}
		return false;
	}

	注解方法
	@Override
	public Set<MethodMetadata> getAnnotatedMethods(String annotationName) {
		Set<MethodMetadata> annotatedMethods = new LinkedHashSet<>(4);
		for (MethodMetadata methodMetadata : this.methodMetadataSet) {
			if (methodMetadata.isAnnotated(annotationName)) {
				annotatedMethods.add(methodMetadata);
			}
		}
		return annotatedMethods;
	}

}


```