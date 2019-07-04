---
layout: post
title:  "Spring源码-BeanNameGenerator BeanName生成器接口"
date:   2019-05-14 23:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Beans
---

* content
{:toc}


BeanNameGenerator用于bean定义生成beanName的策略接口，接口定义如下：

```java
public interface BeanNameGenerator {

	/**
	 * Generate a bean name for the given bean definition.  ：为给定的bean定义生成bean名称
	 * @param definition the bean definition to generate a name for 为其生成名称的bean定义
	 * @param registry the bean definition registry that the given definition 注册支持注册的给定的bean定义
	 * is supposed to be registered with 
	 * @return the generated bean name 生成的bean名称
	 */
	String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry);

}

```

Spring中提供了两种实现：AnnotationBeanNameGenerator和DefaultBeanNameGenerator






## DefaultBeanNameGenerator：默认BeanName生成器

默认BeanNameGenerator接口的实现，委托给BeanDefinitionReaderUtils#generateBeanName(BeanDefinition, BeanDefinitionRegistry)方法，内部执行过程参考：[Spring源码-BeanDefinitionReaderUtils]()

```java
/**
 * Default implementation of the {@link BeanNameGenerator} interface, delegating to
 * {@link BeanDefinitionReaderUtils#generateBeanName(BeanDefinition, BeanDefinitionRegistry)}.
 *
 * @author Juergen Hoeller
 * @since 2.0.3
 */
public class DefaultBeanNameGenerator implements BeanNameGenerator {

	@Override
	public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
		return BeanDefinitionReaderUtils.generateBeanName(definition, registry);
	}

}
```


## AnnotationBeanNameGenerator：注解BeanName生成器

AnnotationBeanNameGenerator为用于使用@Component、@Repository等注解bean类的BeanNameGenerator实现

```java

/**
 * {@link org.springframework.beans.factory.support.BeanNameGenerator} 
 * implementation for bean classes annotated with the
 * {@link org.springframework.stereotype.Component @Component} annotation
 * or with another annotation that is itself annotated with
 * {@link org.springframework.stereotype.Component @Component} as a 元注解
 * meta-annotation. For example, Spring's stereotype annotations (such as
 * {@link org.springframework.stereotype.Repository @Repository}) are
 * themselves annotated with
 * {@link org.springframework.stereotype.Component @Component}.
 *
 * <p>Also supports Java EE 6's {@link javax.annotation.ManagedBean} and 支持java ee6提供的@ManagedBean和JSR-330的@Named
 * JSR-330's {@link javax.inject.Named} annotations, if available. Note that
 * Spring component annotations always override such standard annotations. Spring的component注解总是重写标准的注解
 *
 * <p>If the annotation's value doesn't indicate a bean name, an appropriate 如果注解的value()方法不能指定bean名称，
 * name will be built based on the short name of the class (with the first 一个适当的名称将基于类的短名称构建
 * letter lower-cased). For example:
 *
 * <pre class="code">com.xyz.FooServiceImpl -&gt; fooServiceImpl</pre>
 * @see org.springframework.stereotype.Component#value()
 * @see org.springframework.stereotype.Repository#value()
 * @see org.springframework.stereotype.Service#value()
 * @see org.springframework.stereotype.Controller#value()
 * @see javax.inject.Named#value()
 */
public class AnnotationBeanNameGenerator implements BeanNameGenerator {

	//@Component注解类全名
	private static final String COMPONENT_ANNOTATION_CLASSNAME = "org.springframework.stereotype.Component";


	@Override
	public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
		if (definition instanceof AnnotatedBeanDefinition) { //如果BeanDefinition为AnnotatedBeanDefinition实例时,从注解中确定beanName
			String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition); 
			if (StringUtils.hasText(beanName)) { //不为空时，直接返回
				// Explicit /ɪk'splɪsɪt/ bean name found. 发现显式bean名称
				return beanName;
			}
		}
		// Fallback: generate a unique default bean name. 生成唯一的默认bean名称
		return buildDefaultBeanName(definition, registry);
	}

	/**
	 * Derive a bean name from one of the annotations on the class. 从类上的注释之一派生bean名称
	 * @param annotatedDef the annotation-aware bean definition
	 * @return the bean name, or {@code null} if none is found 为null时为找到
	 */
	@Nullable
	protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) { 
		AnnotationMetadata amd = annotatedDef.getMetadata(); //注解元数据
		Set<String> types = amd.getAnnotationTypes(); //注解类上的所有注解类型名
		String beanName = null;
		for (String type : types) { //遍历注解类型名
			AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type); //从AnnotationMetadata中获取注解名对应的注解属性
			//注解类型是否为给定的组件注解
			if (attributes != null && isStereotypeWithNameValue(type, amd.getMetaAnnotationTypes(type), attributes)) {
				Object value = attributes.get("value"); //获取注解中的value()
				if (value instanceof String) {
					String strVal = (String) value;
					if (StringUtils.hasLength(strVal)) { //不为null时
						if (beanName != null && !strVal.equals(beanName)) {
							throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
									"component names: '" + beanName + "' versus '" + strVal + "'");
						}
						beanName = strVal;
					}
				}
			}
		}
		return beanName;
	}

	/**
	 * Check whether the given annotation is a stereotype that is allowed 检查给定的注解是否是一个原型，通过注解的value（）允许显式组件名
	 * to suggest a component name through its annotation {@code value()}.
	 * @param annotationType the name of the annotation class to check 要检测的注解类的名字
	 * @param metaAnnotationTypes the names of meta-annotations on the given annotation  给定注解的元数据名字
	 * @param attributes the map of attributes for the given annotation 给定注解的属性
	 * @return whether the annotation qualifies as a stereotype with component name 注释是否符合具有组件名称的原型的条件
	 */
	protected boolean isStereotypeWithNameValue(String annotationType,
			Set<String> metaAnnotationTypes, @Nullable Map<String, Object> attributes) {

		boolean isStereotype = annotationType.equals(COMPONENT_ANNOTATION_CLASSNAME) || //注解类名为：org.springframework.stereotype.Component
				metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME) || //元注解类型包含
				annotationType.equals("javax.annotation.ManagedBean") || //@ManagedBeaan
				annotationType.equals("javax.inject.Named"); //@Named

		return (isStereotype && attributes != null && attributes.containsKey("value")); //注解属性中包含value
	}

	/**
	 * Derive a default bean name from the given bean definition.
	 * <p>The default implementation delegates to {@link #buildDefaultBeanName(BeanDefinition)}.
	 * @param definition the bean definition to build a bean name for 要为其构建bean名称的bean定义
	 * @param registry the registry that the given bean definition is being registered with
	 * @return the default bean name (never {@code null})
	 */
	protected String buildDefaultBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
		return buildDefaultBeanName(definition);
	}

	/**
	 * Derive a default bean name from the given bean definition. 从给定的BeanDefinition派生一个默认bean名称
	 * <p>The default implementation simply builds a decapitalized version 默认的实现简单的构建一个头版本的短类名
	 * of the short class name: e.g. "mypackage.MyJdbcDao" -> "myJdbcDao".
	 * <p>Note that inner classes will thus have names of the form 内部类如此： outerClassName.InnerClassName
	 * "outerClassName.InnerClassName", which because of the period in the 
	 * name may be an issue 问题 if you are autowiring by name. 如果通过name自动注入
	 * @param definition the bean definition to build a bean name for
	 * @return the default bean name (never {@code null})
	 */
	protected String buildDefaultBeanName(BeanDefinition definition) {
		String beanClassName = definition.getBeanClassName(); //获取类的名称
		Assert.state(beanClassName != null, "No bean class name set");
		String shortClassName = ClassUtils.getShortName(beanClassName); //获取类的短名称
		return Introspector.decapitalize(shortClassName);//内省调整短类名
	}

}
```