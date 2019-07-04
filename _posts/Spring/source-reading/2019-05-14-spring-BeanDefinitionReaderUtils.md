---
layout: post
title:  "Spring源码-BeanDefinitionReaderUtils"
date:   2019-05-08 23:57:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}

BeanDefinitionReaderUtils：对于读取bean定义非常有用，主要用于内部使用。包含如下静态方法：

- AbstractBeanDefinition createBeanDefinition(@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader)：根据parentName或者class name AbstractBeanDefinition对象

- String generateBeanName(BeanDefinition beanDefinition, BeanDefinitionRegistry registry)、String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean)：生成Bean Name

- registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)：根据definitionHolder将Bean Definition注册到BeanDefinitionRegistry中

- registerWithGeneratedName(AbstractBeanDefinition definition, BeanDefinitionRegistry registry):根据AbstractBeanDefinition生成beanName，且注册到BeanDefinitionRegistry中







```java
/**
 * Utility methods that are useful for bean definition reader implementations.
 * Mainly intended for internal use.
 *
 * @since 1.1
 * @see PropertiesBeanDefinitionReader：属性Bean定义读取器
 * @see org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader 默认Bean定义文档读取器
 */
public class BeanDefinitionReaderUtils {

	/**
	 * Separator for generated bean names 用于生成bean名称的分隔符. If a class name or parent name is not 如果一个类或者父类的名字不是唯一的，将使用#缀加，直到变为唯一
	 * unique, "#1", "#2" etc will be appended, until the name becomes unique.
	 */
	public static final String GENERATED_BEAN_NAME_SEPARATOR = BeanFactoryUtils.GENERATED_BEAN_NAME_SEPARATOR; //"#


	/**
	 * Create a new GenericBeanDefinition for the given parent name and class name, 为给定的parent bean和bean 创建一个新的GenericBeanDefinition
	 * eagerly loading the bean class if a ClassLoader has been specified. 如果指定了一个ClassLoader，立即加载bean类
	 * @param parentName the name of the parent bean, if any 父bean的名称
	 * @param className the name of the bean class, if any bean类的名称
	 * @param classLoader the ClassLoader to use for loading bean classes ：用于加载bean类的类加载器
	 * (can be {@code null} to just register bean classes by name) 
	 * @return the bean definition
	 * @throws ClassNotFoundException if the bean class could not be loaded
	 */
	public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition(); //创建GenericBeanDefinition对象
		bd.setParentName(parentName); //设置父类名称，可为null
		if (className != null) { 
			if (classLoader != null) { //类名不为null时，且类加载器不为null时，返回类名对应的类
				bd.setBeanClass(ClassUtils.forName(className, classLoader)); //设置bean 对应的类
			}
			else {
				bd.setBeanClassName(className);//设置bean类的名称
			}
		}
		return bd;
	}

	/**
	 * Generate a bean name for the given top-level bean definition, 为给定的顶级bean定义生成bean名称
	 * unique within the given bean factory. 在给定的bean工厂中是惟一的
	 * @param beanDefinition the bean definition to generate a bean name for 用于生成bean名称的bean定义
	 * @param registry the bean factory that the definition is going to be 定义要注册的bean工厂(用于检查现有bean名称)
	 * registered with (to check for existing bean names)
	 * @return the generated bean name
	 * @throws BeanDefinitionStoreException if no unique name can be generated
	 * for the given bean definition 如果生成的bean name不唯一时，抛出BeanDefinitionStoreException异常
	 * @see #generateBeanName(BeanDefinition, BeanDefinitionRegistry, boolean)
	 */
	public static String generateBeanName(BeanDefinition beanDefinition, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		return generateBeanName(beanDefinition, registry, false);
	}

	/**
	 * Generate a bean name for the given bean definition, unique within the
	 * given bean factory.
	 * @param definition the bean definition to generate a bean name for
	 * @param registry the bean factory that the definition is going to be
	 * registered with (to check for existing bean names)
	 * @param isInnerBean whether the given bean definition will be registered 是否给定的bean定义作为一个内部bean或者顶层bean注册
	 * as inner bean 内部bean or as top-level bean 顶层bean(allowing for special name generation 允许生成特殊的名称
	 * for inner beans versus top-level beans) 用于内部bean和顶级bean
	 * @return the generated bean name
	 * @throws BeanDefinitionStoreException if no unique name can be generated
	 * for the given bean definition
	 */
	public static String generateBeanName(
			BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean)
			throws BeanDefinitionStoreException {

		String generatedBeanName = definition.getBeanClassName(); //获取bean类的名称
		if (generatedBeanName == null) { 
			//为null时，从bean的parentName属性获取
			if (definition.getParentName() != null) {
				generatedBeanName = definition.getParentName() + "$child"; 
			}
			else if (definition.getFactoryBeanName() != null) { //工厂beanName不为null时
				generatedBeanName = definition.getFactoryBeanName() + "$created";
			}
		}
		if (!StringUtils.hasText(generatedBeanName)) {
			throw new BeanDefinitionStoreException("Unnamed bean definition specifies neither " +
					"'class' nor 'parent' nor 'factory-bean' - can't generate bean name");
		}
		//以上步骤主要获取顶层bean名，并缀加不同的标记父"$child"或者"$created"
		String id = generatedBeanName;
		if (isInnerBean) { //内部bean
			// Inner bean: generate identity hashcode suffix. 生成标识hashcode后缀 // ObjectUtils.getIdentityHexString()生成bean实例的16进制hashcode值
			id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + ObjectUtils.getIdentityHexString(definition);
		}
		else {
			// Top-level bean: use plain class name. 用普通类名
			// Increase counter until the id is unique. 增加计数器，直到id是惟一的
			int counter = -1;
			while (counter == -1 || registry.containsBeanDefinition(id)) { //bean的注册器中使用包含该beanName
				counter++;
				id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + counter;
			}
		}
		return id;
	}

	/**
	 * Register the given bean definition with the given bean factory. 向给定的bean工厂注册给定的bean定义
	 * @param definitionHolder the bean definition including name and aliases definitionHolder 包含bean定义的名字和别名
	 * @param registry the bean factory to register with
	 * @throws BeanDefinitionStoreException if registration failed
	 */
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name. 在主名称下注册bean
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any. //注册bean的别名
		String[] aliases = definitionHolder.getAliases();//
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}

	/**
	 * Register the given bean definition with a generated name, 用于自动生成的name注册给定的bean定义
	 * unique within the given bean factory. 在给定的bean工厂中时唯一的
	 * @param definition the bean definition to generate a bean name for
	 * @param registry the bean factory to register with
	 * @return the generated bean name
	 * @throws BeanDefinitionStoreException if no unique name can be generated
	 * for the given bean definition or the definition cannot be registered
	 */
	public static String registerWithGeneratedName(
			AbstractBeanDefinition definition, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		String generatedName = generateBeanName(definition, registry, false); //生成beanName
		registry.registerBeanDefinition(generatedName, definition); //注册bean定义
		return generatedName; //返回生成的名称
	}

}


```