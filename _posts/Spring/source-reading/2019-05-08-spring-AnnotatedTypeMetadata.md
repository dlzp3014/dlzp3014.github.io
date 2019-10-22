---
layout: post
title:  "Spring源码-AnnotatedTypeMetadata注解类型元数据接口"
date:   2019-05-08 21:36:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

AnnotatedTypeMetadata注解类型元数据接口用于定义访问特定类型的AnnotationMetadata【具体参看：[Spring源码-AnnotationMetadata注解元数据接口](/2019/10/19/spring-AnnotationMetadata/)】类或者MethodMetadata【具体参看：[Spring源码-MethodMetadata方法元数据接口](/2019/10/19/spring-MethodMetadata/)】方法注解，而不需要加载类。此接口由Spring 4.0版本添加，统一了AnnotationMetadata和MethodMetadata接口注解的访问。

【Note】：AnnotatedTypeMetadata接口中定义的`Map<String, Object> getAnnotationAttributes()`方法中返回Map<String ,Object> 类型具体为AnnotationAttributes子类，代表注解属性键值对，具体请参考：[Spring源码-AnnotationAttributes注解属性类型安全模式](/2019/10/21/spring-AnnotationAttributes/)
源代码如下：






```java
/**
 * Defines access to the annotations of a specific type ({@link AnnotationMetadata class}
 * or {@link MethodMetadata method}), in a form that does not necessarily require the
 * class-loading.
 *
 * @since 4.0
 * @see AnnotationMetadata
 * @see MethodMetadata
 */
public interface AnnotatedTypeMetadata {

	/**
	 * Determine whether the underlying element has an annotation or meta-annotation
	 * of the given type defined. 确认给定的类定义底层元素是否有注解或者源注解
	 * <p>If this method returns {@code true}, then
	 * {@link #getAnnotationAttributes} will return a non-null Map.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for 要查找的权限类名注解
	 * @return whether a matching annotation is defined
	 */
	boolean isAnnotated(String annotationName);

	/** 检索给定给下注解的属性，如果有的话，还要考虑重写复合注释的属性
	 * Retrieve the attributes of the annotation of the given type, if any (i.e. if
	 * defined on the underlying element, as direct annotation or meta-annotation),
	 * also taking attribute overrides on composed annotations into account. 
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for
	 * @return a Map of attributes, with the attribute name as key (e.g. "value")
	 * and the defined attribute value as Map value. This return value will be
	 * {@code null} if no matching annotation is defined.
	 */
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName);

	/**
	 * Retrieve the attributes of the annotation of the given type, if any (i.e. if
	 * defined on the underlying element, as direct annotation or meta-annotation),
	 * also taking attribute overrides on composed annotations into account.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for  
	 * @param classValuesAsString whether to convert class references to String 是否将类引用转换为字符串在返回的映射中作为值公开的类名
	 * class names for exposure as values in the returned Map, instead of Class
	 * references which might potentially have to be loaded first 
	 * @return a Map of attributes, with the attribute name as key (e.g. "value")
	 * and the defined attribute value as Map value. This return value will be
	 * {@code null} if no matching annotation is defined.
	 */
	@Nullable
	Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString);

	/**
	 * Retrieve all attributes of all annotations of the given type, if any (i.e. if
	 * defined on the underlying element, as direct annotation or meta-annotation).
	 * Note that this variant does <i>not</i> take 不考虑重写attribute overrides into account.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for
	 * @return a MultiMap of attributes, with the attribute name as key (e.g. "value")
	 * and a list of the defined attribute values as Map value. This return value will
	 * be {@code null} if no matching annotation is defined.
	 * @see #getAllAnnotationAttributes(String, boolean)
	 */
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName);

	/**
	 * Retrieve all attributes of all annotations of the given type, if any (i.e. if
	 * defined on the underlying element, as direct annotation or meta-annotation).
	 * Note that this variant does <i>not</i> take attribute overrides into account.
	 * @param annotationName the fully qualified class name of the annotation
	 * type to look for
	 * @param classValuesAsString  whether to convert class references to String
	 * @return a MultiMap of attributes, with the attribute name as key (e.g. "value")
	 * and a list of the defined attribute values as Map value. This return value will
	 * be {@code null} if no matching annotation is defined.
	 * @see #getAllAnnotationAttributes(String)
	 */
	@Nullable
	MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString);

}

```

