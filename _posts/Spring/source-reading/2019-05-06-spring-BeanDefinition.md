---
layout: post
title:  "Spring源码-BeanDefinition Bean定义"
date:   2019-05-07 22:26:00
categories: Spring 
tags: Spring-Source-Reading Spring-Beans
---

* content
{:toc}

BeanDefinition接口描述了一个Bean的实例，包含属性值、构造参数值、以及通过具体实现进一步的信息。BeanDefinition 只是一个最小的接口，主要的意图是允许BeanFactoryPostProcessor(Bean工厂后置处理器)，如PropertyPlaceholderConfigurer(属性占位符配置)，内省、修改属性值、其他bean元数据。类结构如下：

![](/img/post.img/spring/BeanDefinition.png)








## BeanDefinition：Bean定义接口

BeanDefinition接口继承了AttributeAccessor属性访问器接口和Bean元数据元素接口

```java
/* @see ConfigurableListableBeanFactory#getBeanDefinition 获取bean的定义
 * @see org.springframework.beans.factory.support.RootBeanDefinition
 * @see org.springframework.beans.factory.support.ChildBeanDefinition
 */
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	/**
	 * Scope identifier for the standard singleton scope: "singleton". 标准单例域标识
	 * <p>Note that extended bean factories might support further scopes.
	 * @see #setScope
	 */
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON; //

	/**
	 * Scope identifier for the standard prototype scope: "prototype". 标准原型域标识
	 * <p>Note that extended bean factories might support further scopes.
	 * @see #setScope
	 */
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;


	/**
	 * Role hint indicating that a {@code BeanDefinition} is a major part  暗示表明 BeanDefinition是application的主要部分
	 * of the application. Typically corresponds to a user-defined bean. 通常对应于用户定义的bean
	 */
	int ROLE_APPLICATION = 0;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is a supporting 一个较大部分配置支持的部分
	 * part of some larger configuration, typically an outer 
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 * {@code SUPPORT} beans are considered important enough to be aware beans被认为是非常重要的，当您更仔细地查看某个特定对象时，应该注意它
	 * of when looking more closely at a particular
	 * {@link org.springframework.beans.factory.parsing.ComponentDefinition},
	 * but not when looking at the overall configuration of an application. 但在查看应用程序的总体配置时就不需要了
	 */
	int ROLE_SUPPORT = 1;

	/**
	 * Role hint indicating that a {@code BeanDefinition} is providing an
	 * entirely 完全背景 background role and has no relevance to the end-user 与最终用户无关. This hint is
	 * used when registering beans that are completely part of the internal workings 注册完全属于内部工作的一部分的bean
	 * of a {@link org.springframework.beans.factory.parsing.ComponentDefinition}.
	 */
	int ROLE_INFRASTRUCTURE = 2;


	// Modifiable attributes  可修改的属性

	/**
	 * Set the name of the parent definition of this bean definition, if any. 设置此bean定义的父定义的名称
	 */
	void setParentName(@Nullable String parentName);

	/**
	 * Return the name of the parent definition of this bean definition, if any.
	 */
	@Nullable
	String getParentName();

	/**
	 * Specify the bean class name of this bean definition. 指定这个bean定义的bean类名字
	 * <p>The class name can be modified during bean factory post-processing, 类名可以在bean工厂的后处理过程中修改
	 * typically replacing the original class name with a parsed variant of it. 通常用已解析的类名变体替换原来的类名
	 * @see #setParentName
	 * @see #setFactoryBeanName
	 * @see #setFactoryMethodName
	 */
	void setBeanClassName(@Nullable String beanClassName);

	/**
	 * Return the current bean class name of this bean definition. 返回当前bean类的名字
	 * <p>Note that this does not have to be the actual class name used at runtime, in 注意：返回的结果并不一定是运行时类的实际名字
	 * case of a child definition overriding/inheriting the class name from its parent. 子bean 定义可重新/继承类的名字
	 * Also, this may just be the class that a factory method is called on 工厂方法调用, or it may 
	 * even be empty in case of a factory bean reference that a method is called on.
	 * Hence, do <i>not</i> consider 不要认为认为 this to be the definitive bean type at runtime but 在运行时bean最后的类型
	 * rather only use it for parsing purposes at the individual bean definition level. 相反的只是使用它解析单个bean定义的等级为目的
	 * @see #getParentName()
	 * @see #getFactoryBeanName()
	 * @see #getFactoryMethodName()
	 */
	@Nullable
	String getBeanClassName();

	/**
	 * Override the target scope of this bean, specifying a new scope name. 重写bean的域，指定一个新的域名
	 * @see #SCOPE_SINGLETON
	 * @see #SCOPE_PROTOTYPE
	 */
	void setScope(@Nullable String scope);

	/**
	 * Return the name of the current target scope for this bean,
	 * or {@code null} if not known yet.
	 */
	@Nullable
	String getScope();

	/**
	 * Set whether this bean should be lazily initialized. 设置bean是否应该延迟初始化
	 * <p>If {@code false}, the bean will get instantiated on startup by bean false时，bean启动时通过bean工厂执行迫切初始化单例时，将其实例化
	 * factories that perform eager initialization of singletons.
	 */
	void setLazyInit(boolean lazyInit);

	/**
	 * Return whether this bean should be lazily initialized, i.e. not
	 * eagerly instantiated 急切地实例化 on startup. Only applicable to a singleton bean. 仅适用于单例bean
	 */
	boolean isLazyInit();

	/**
	 * Set the names of the beans that this bean depends on being initialized. 设置此bean依赖于初始化的bean的名称
	 * The bean factory will guarantee 确保 that these beans get initialized first. 其他bean先初始化
	 */
	void setDependsOn(@Nullable String... dependsOn);

	/**
	 * Return the bean names that this bean depends on.
	 */
	@Nullable
	String[] getDependsOn();

	/**
	 * Set whether this bean is a candidate 候选 for getting autowired into some other bean. 用于获取自动装备到其他bean中
	 * <p>Note that this flag is designed to only affect type-based autowiring. 标志设计， 只影响基于类型的自动装配
	 * It does not affect explicit references by name 它不影响按名称显式引用, which will get resolved 得到解决 even 
	 * if the specified bean is not marked as an autowire candidate. As a consequence(因此), 即使指定的bean不是一个标记了自动装配候选
	 * autowiring by name will nevertheless inject a bean if the name matches.如果名称匹配， 根据名称自动装配注入
	 */
	void setAutowireCandidate(boolean autowireCandidate); //自动装配的候选：仅影响类型，而不适用于名称

	/**
	 * Return whether this bean is a candidate for getting autowired into some other bean.  返回这个bean是否是自动生成其他bean的候选bean
	 */
	boolean isAutowireCandidate();

	/**
	 * Set whether this bean is a primary autowire candidate. 主要的自动装配候选
	 * <p>If this value is {@code true} for exactly . 恰好地 one bean among multiple 多个匹配候选项中的一个bean
	 * matching candidates, it will serve as a tie-breaker. 这将起到决定性作用
	 */
	void setPrimary(boolean primary);

	/**
	 * Return whether this bean is a primary autowire candidate.
	 */
	boolean isPrimary();

	/**
	 * Specify the factory bean to use, if any. 指定要使用的工厂bean
	 * This the name of the bean to call the specified factory method on. 这是要调用指定的工厂方法的bean的名称
	 * @see #setFactoryMethodName
	 */
	void setFactoryBeanName(@Nullable String factoryBeanName);

	/**
	 * Return the factory bean name, if any.
	 */
	@Nullable
	String getFactoryBeanName();

	/**
	 * Specify a factory method, if any. This method will be invoked with 指定工厂方法，这个方法将通过构造参数调用
	 * constructor arguments, or with no arguments if none are specified. 或者没有指定参数
	 * The method will be invoked on the specified factory bean, if any, 方法将在指定的工厂bean上调用
	 * or otherwise as a static method on the local bean class. 或者作为本地bean类上的静态方法
	 * @see #setFactoryBeanName
	 * @see #setBeanClassName
	 */
	void setFactoryMethodName(@Nullable String factoryMethodName);

	/**
	 * Return a factory method, if any.
	 */
	@Nullable
	String getFactoryMethodName();

	/**
	 * Return the constructor argument values for this bean. 构造函数参数值
	 * <p>The returned instance can be modified during bean factory post-processing. 返回的实例可以在bean工厂的后处理过程中修改
	 * @return the ConstructorArgumentValues object (never {@code null})
	 */
	ConstructorArgumentValues getConstructorArgumentValues();

	/**
	 * Return if there are constructor argument values defined for this bean. 默认方法，是否有构造参数值，没有时返回true
	 * @since 5.0.2
	 */
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	/**
	 * Return the property values to be applied to a new instance of the bean. 返回要应用于bean的新实例的属性值
	 * <p>The returned instance can be modified during bean factory post-processing. 返回的实例可以在bean工厂的后处理过程中修改
	 * @return the MutablePropertyValues object (never {@code null})
	 */
	MutablePropertyValues getPropertyValues();

	/**
	 * Return if there are property values values defined for this bean.再bean定义中是否有属性值
	 * @since 5.0.2
	 */
	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}


	// Read-only attributes 只读属性

	/** 
	 * Return whether this a <b>Singleton</b>, with a single, shared instance 是否为单例
	 * returned on all calls.
	 * @see #SCOPE_SINGLETON
	 */
	boolean isSingleton();

	/**
	 * Return whether this a <b>Prototype</b>, with an independent instance 是否为原型
	 * returned for each call.
	 * @since 3.0
	 * @see #SCOPE_PROTOTYPE
	 */
	boolean isPrototype();

	/**
	 * Return whether this bean is "abstract" 抽象, that is, not meant to be instantiated.  不打算实例化
	 */
	boolean isAbstract();

	/**
	 * Get the role hint 提示作用 for this {@code BeanDefinition}. The role hint
	 * provides the frameworks as well as tools with an indication of 提示作用提供框架和工具，指示BeanDefinition特别的角色和重要性
	 * the role and importance of a particular {@code BeanDefinition}.
	 * @see #ROLE_APPLICATION
	 * @see #ROLE_SUPPORT
	 * @see #ROLE_INFRASTRUCTURE
	 */
	int getRole();

	/**
	 * Return a human-readable description 可读的描述 of this bean definition.
	 */
	@Nullable
	String getDescription();

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors). 以便在出现错误时显示上下文
	 */
	@Nullable
	String getResourceDescription();

	/**
	 * Return the originating BeanDefinition 原始, or {@code null} if none.
	 * Allows for retrieving the decorated bean definition, 运行检索装饰的bean定义if any. 
	 * <p>Note that this method returns the immediate originator 直接的发起者. Iterate through the
	 * originator chain to find the original BeanDefinition as defined by the user. 通过迭代原始连接查找通过用户定义的原始BeanDefinition，
	 */
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}

```

## AnnotatedBeanDefinition：注解Bean定义接口



```java
/**
 * Extended {@link org.springframework.beans.factory.config.BeanDefinition} 扩展BeanDefinition接口
 * interface that exposes {@link org.springframework.core.type.AnnotationMetadata} 暴露关于它的bean类的AnnotationMetadata注解元数据
 * about its bean class - without requiring the class to be loaded yet. 还不需要加载类
 *
 * @author Juergen Hoeller
 * @since 2.5
 * @see AnnotatedGenericBeanDefinition 通用注解Bean定义
 * @see org.springframework.core.type.AnnotationMetadata 注解元数据
 */
public interface AnnotatedBeanDefinition extends BeanDefinition {

	/**
	 * Obtain the annotation metadata (as well as basic class metadata) 从当前bean定义的bean类获取注解元数据(以及基本类元数据)
	 * for this bean definition's bean class.
	 * @return the annotation metadata object (never {@code null})
	 */
	AnnotationMetadata getMetadata();

	/**
	 * Obtain metadata for this bean definition's factory method, if any. 获取bean定义的工厂方法的元数据，
	 * @return the factory method metadata, or {@code null} if none 工厂方法元数据
	 * @since 4.1.1
	 */
	@Nullable
	MethodMetadata getFactoryMethodMetadata();

}
```

## AbstractBeanDefinition：抽象Bean定义实现

AbstractBeanDefinition 继承了BeanMetadataAttributeAccessor(Bean元数据属性访问器)，而BeanMetadataAttributeAccessor继承AttributeAccessorSupport(属性访问器支持)，实现了BeanMetadataElement(Bean元数据元素)接口，AttributeAccessorSupport实现了AttributeAccessor接口，因此AbstractBeanDefinition仅需要实现BeanDefinition定义的接口即可，结构如下：

![](/img/post.img/spring/AbstractBeanDefinition.png)

- AbstractBeanDefinition类描述：

```java
/**
 * Base class for concrete, full-fledged {@link BeanDefinition} classes,
 * factoring out common properties of {@link GenericBeanDefinition}, 
 * {@link RootBeanDefinition}, and {@link ChildBeanDefinition}.
 *
 * <p>The autowire constants match the ones defined in the  自动装配常数匹配在AutowireCapableBeanFactory接口中定义
 * {@link org.springframework.beans.factory.config.AutowireCapableBeanFactory}
 * interface.
 *
 * @see GenericBeanDefinition
 * @see RootBeanDefinition
 * @see ChildBeanDefinition
 */
@SuppressWarnings("serial")
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {
}
```

### AbstractBeanDefinition提供的常量

```java
	/**
	 * Constant for the default scope name: {@code ""}, equivalent to singleton 默认域名为空，等价于单例状态
	 * status unless overridden from a parent bean definition (if applicable). 除非从父bean定义重写
	 */
	public static final String SCOPE_DEFAULT = "";

	/**
	 * Constant that indicates no autowiring at all. 指示没有自动装配
	 * @see #setAutowireMode
	 */
	public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;

	/**
	 * Constant that indicates autowiring bean properties by name. 通过名称指示自动装配bean属性
	 * @see #setAutowireMode
	 */
	public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;

	/**
	 * Constant that indicates autowiring bean properties by type. 通过类型指示自动装配bean属性
	 * @see #setAutowireMode
	 */
	public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;

	/**
	 * Constant that indicates autowiring a constructor. 指示自动装配构造函数
	 * @see #setAutowireMode
	 */
	public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;

	/**
	 * Constant that indicates determining an appropriate autowire strategy 通过内省Bean类，确认适合的自动装配策略，从Spring 3.x后已经废弃
	 * through introspection of the bean class.
	 * @see #setAutowireMode
	 * @deprecated as of Spring 3.0: If you are using mixed autowiring strategies,
	 * use annotation-based autowiring for clearer demarcation of autowiring needs.
	 */
	@Deprecated
	public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;

	/**
	 * Constant that indicates no dependency check at all.指示没有依赖检查
	 * @see #setDependencyCheck
	 */
	public static final int DEPENDENCY_CHECK_NONE = 0;

	/**
	 * Constant that indicates dependency checking for object references 指示依赖检测对象引用.
	 * @see #setDependencyCheck
	 */
	public static final int DEPENDENCY_CHECK_OBJECTS = 1;

	/**
	 * Constant that indicates dependency checking for "simple" properties. 简单属性
	 * @see #setDependencyCheck
	 * @see org.springframework.beans.BeanUtils#isSimpleProperty
	 */
	public static final int DEPENDENCY_CHECK_SIMPLE = 2;

	/**
	 * Constant that indicates dependency checking for all properties 检查所有属性
	 * (object references as well as "simple" properties).  对象引用以及“简单”属性
	 * @see #setDependencyCheck
	 */
	public static final int DEPENDENCY_CHECK_ALL = 3;

	/**
	 * Constant that indicates the container should attempt to infer 推断 the
	 * {@link #setDestroyMethodName destroy method name 销毁方法名} for a bean as opposed 相反 to
	 * explicit 明确的 specification of a method name. The value {@value} is specifically @Value明确地目的在于
	 * designed to include characters otherwise illegal in a method name 方法名不合法, ensuring
	 * no possibility of collisions 冲突 with legitimately 合理地 named methods having the same
	 * name.
	 * <p>Currently, the method names detected during destroy method inference 在销毁方法期间检测到的方法名称推理为close和shutdown
	 * are "close" and "shutdown", if present on the specific bean class. 如果出现在特定的bean类上
	 */
	public static final String INFER_METHOD = "(inferred)";
```

### AbstractBeanDefinition中的成员变量

```java
	@Nullable
	private volatile Object beanClass; //内存中可见，bean类代表的对象

	@Nullable
	private String scope = SCOPE_DEFAULT; //bean所在域，默认为"",单例

	private boolean abstractFlag = false; //抽象标志，默认false

	private boolean lazyInit = false; //延迟初始化，默认false

	private int autowireMode = AUTOWIRE_NO; //自动装配默认，默认不进行自动装配

	private int dependencyCheck = DEPENDENCY_CHECK_NONE; //依赖检查，默认不进行检检查

	@Nullable
	private String[] dependsOn; //依赖的Bean 名称

	private boolean autowireCandidate = true; //自动装配候选，默认true

	private boolean primary = false; //主要的自动装配候选，默认false

	//修饰符：key为AutowireCandidateQualifier#getTypeName :(the name of the annotation type 注解类型的名字)
	private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

	@Nullable
	private Supplier<?> instanceSupplier; //实例的提供者

	private boolean nonPublicAccessAllowed = true; //是否允许访问没有public修饰的构造函数和方法

	private boolean lenientConstructorResolution = true; //宽松的构造函数解析

	@Nullable
	private String factoryBeanName;  //工厂Bean名，调用指定的工厂方法的bean的名称

	@Nullable
	private String factoryMethodName; //工厂方法名

	@Nullable
	private ConstructorArgumentValues constructorArgumentValues; //构造函数参数值

	@Nullable
	private MutablePropertyValues propertyValues;  //可变属性值

	@Nullable
	private MethodOverrides methodOverrides; //方法重写

	@Nullable
	private String initMethodName; //初始化方法名

	@Nullable
	private String destroyMethodName; //销毁方法名

	private boolean enforceInitMethod = true; //指示所配置的init方法是否是默认值

	private boolean enforceDestroyMethod = true; 

	private boolean synthetic = false; // 合成,不是由应用自身定义
 
	private int role = BeanDefinition.ROLE_APPLICATION; //用户定义的bean

	@Nullable
	private String description; //

	@Nullable
	private Resource resource; //引用BeanDefinitionResource

```
### AbstractBeanDefinition提供的构造函数

```java
/**
 * Create a new AbstractBeanDefinition with default settings. 创建新的AbstractBeanDefinition，使用默认的设置
 */
protected AbstractBeanDefinition() {
	this(null, null);
}

/**
 * Create a new AbstractBeanDefinition with the given 使用给定的构造参数值和属性值创建AbstractBeanDefinition，两者都可能为null
 * constructor argument values and property values.
 */
protected AbstractBeanDefinition(@Nullable ConstructorArgumentValues cargs, @Nullable MutablePropertyValues pvs) {
	this.constructorArgumentValues = cargs;
	this.propertyValues = pvs;
}

/**
 * Create a new AbstractBeanDefinition as a deep copy of the given 深度copy给定的bean定义
 * bean definition.
 * @param original the original bean definition to copy from
 */
protected AbstractBeanDefinition(BeanDefinition original) {
	setParentName(original.getParentName());
	setBeanClassName(original.getBeanClassName());
	setScope(original.getScope());
	setAbstract(original.isAbstract());
	setLazyInit(original.isLazyInit());
	setFactoryBeanName(original.getFactoryBeanName());
	setFactoryMethodName(original.getFactoryMethodName());
	setRole(original.getRole());
	setSource(original.getSource());
	copyAttributesFrom(original);

	if (original instanceof AbstractBeanDefinition) { //如果为AbstractBeanDefinition
		AbstractBeanDefinition originalAbd = (AbstractBeanDefinition) original;
		if (originalAbd.hasBeanClass()) { //有指定bean类
			setBeanClass(originalAbd.getBeanClass());
		}
		if (originalAbd.hasConstructorArgumentValues()) { //有构造参数值
			setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
		}
		if (originalAbd.hasPropertyValues()) { //有属性值
			setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
		}
		if (originalAbd.hasMethodOverrides()) { //有重写的方法
			setMethodOverrides(new MethodOverrides(originalAbd.getMethodOverrides()));
		}
		setAutowireMode(originalAbd.getAutowireMode());
		setDependencyCheck(originalAbd.getDependencyCheck());
		setDependsOn(originalAbd.getDependsOn());
		setAutowireCandidate(originalAbd.isAutowireCandidate());
		setPrimary(originalAbd.isPrimary());
		copyQualifiersFrom(originalAbd);
		setInstanceSupplier(originalAbd.getInstanceSupplier());
		setNonPublicAccessAllowed(originalAbd.isNonPublicAccessAllowed());
		setLenientConstructorResolution(originalAbd.isLenientConstructorResolution());
		setInitMethodName(originalAbd.getInitMethodName());
		setEnforceInitMethod(originalAbd.isEnforceInitMethod());
		setDestroyMethodName(originalAbd.getDestroyMethodName());
		setEnforceDestroyMethod(originalAbd.isEnforceDestroyMethod());
		setSynthetic(originalAbd.isSynthetic());
		setResource(originalAbd.getResource());
	}
	else {
		setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
		setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
		setResourceDescription(original.getResourceDescription());
	}
}

```

### AbstractBeanDefinition提供的核心方法：

- overrideFrom(BeanDefinition other): 从给定的bean定义中覆盖此bean定义中的设置

```java
	/**
	 * Override settings in this bean definition (presumably a copied parent  可能从一个父-子继承关系复制父bean
	 * from a parent-child inheritance relationship) from the given bean
	 * definition (presumably the child).
	 * <ul>
	 * <li>Will override beanClass if specified in the given bean definition. 如果在给定的bean定义中指定，将覆盖bean类
	 * <li>Will always take {@code abstract}, {@code scope},
	 * {@code lazyInit}, {@code autowireMode}, {@code dependencyCheck},
	 * and {@code dependsOn} from the given bean definition.
	 * <li>Will add {@code constructorArgumentValues}, {@code propertyValues},
	 * {@code methodOverrides} from the given bean definition to existing ones. 存在添加 constructorArgumentValues 、propertyValues
	 * <li>Will override {@code factoryBeanName}, {@code factoryMethodName},
	 * {@code initMethodName}, and {@code destroyMethodName} if specified
	 * in the given bean definition.
	 * </ul>
	 */
	public void overrideFrom(BeanDefinition other) {
		if (StringUtils.hasLength(other.getBeanClassName())) { //beanClassName存在
			setBeanClassName(other.getBeanClassName());
		}
		if (StringUtils.hasLength(other.getScope())) { //域存在
			setScope(other.getScope());
		}
		setAbstract(other.isAbstract()); //抽象
		setLazyInit(other.isLazyInit());//延迟初始化
		if (StringUtils.hasLength(other.getFactoryBeanName())) {
			setFactoryBeanName(other.getFactoryBeanName());
		}
		if (StringUtils.hasLength(other.getFactoryMethodName())) {
			setFactoryMethodName(other.getFactoryMethodName());
		}
		setRole(other.getRole());
		setSource(other.getSource());
		copyAttributesFrom(other);

		if (other instanceof AbstractBeanDefinition) {
			AbstractBeanDefinition otherAbd = (AbstractBeanDefinition) other;
			if (otherAbd.hasBeanClass()) {
				setBeanClass(otherAbd.getBeanClass());
			}
			if (otherAbd.hasConstructorArgumentValues()) { //存在构造函数值
				getConstructorArgumentValues().addArgumentValues(other.getConstructorArgumentValues());
			}
			if (otherAbd.hasPropertyValues()) {//存在属性值
				getPropertyValues().addPropertyValues(other.getPropertyValues());
			}
			if (otherAbd.hasMethodOverrides()) {//方法重写
				getMethodOverrides().addOverrides(otherAbd.getMethodOverrides());
			}
			setAutowireMode(otherAbd.getAutowireMode()); //自动装配模式
			setDependencyCheck(otherAbd.getDependencyCheck());
			setDependsOn(otherAbd.getDependsOn());
			setAutowireCandidate(otherAbd.isAutowireCandidate());
			setPrimary(otherAbd.isPrimary());
			copyQualifiersFrom(otherAbd);
			setInstanceSupplier(otherAbd.getInstanceSupplier());
			setNonPublicAccessAllowed(otherAbd.isNonPublicAccessAllowed());
			setLenientConstructorResolution(otherAbd.isLenientConstructorResolution());
			if (otherAbd.getInitMethodName() != null) { //初始化方法
				setInitMethodName(otherAbd.getInitMethodName());
				setEnforceInitMethod(otherAbd.isEnforceInitMethod());
			}
			if (otherAbd.getDestroyMethodName() != null) { //bean销毁方法
				setDestroyMethodName(otherAbd.getDestroyMethodName());
				setEnforceDestroyMethod(otherAbd.isEnforceDestroyMethod());
			}
			setSynthetic(otherAbd.isSynthetic());
			setResource(otherAbd.getResource());
		}
		else {
			getConstructorArgumentValues().addArgumentValues(other.getConstructorArgumentValues());
			getPropertyValues().addPropertyValues(other.getPropertyValues());
			setResourceDescription(other.getResourceDescription());
		}
	}
```

- applyDefaults(BeanDefinitionDefaults defaults)

```java
/**
 * Apply the provided default values to this bean. 提供的默认值应用于此bean
 * @param defaults the defaults to apply 要应用的默认值
 */
public void applyDefaults(BeanDefinitionDefaults defaults) {
	setLazyInit(defaults.isLazyInit());
	setAutowireMode(defaults.getAutowireMode());
	setDependencyCheck(defaults.getDependencyCheck());
	setInitMethodName(defaults.getInitMethodName());
	setEnforceInitMethod(false);
	setDestroyMethodName(defaults.getDestroyMethodName());
	setEnforceDestroyMethod(false);
}

```

BeanDefinitionDefaults:一个简单的BeanDefinition属性默认值的持有者

```java
public class BeanDefinitionDefaults {

	private boolean lazyInit;

	private int dependencyCheck = AbstractBeanDefinition.DEPENDENCY_CHECK_NONE; //不检查

	private int autowireMode = AbstractBeanDefinition.AUTOWIRE_NO; //不自动装配

	@Nullable
	private String initMethodName; //初始化方法

	@Nullable
	private String destroyMethodName; //bean销毁后回调的方法


	public void setLazyInit(boolean lazyInit) {
		this.lazyInit = lazyInit;
	}

	public boolean isLazyInit() {
		return this.lazyInit;
	}

	public void setDependencyCheck(int dependencyCheck) {
		this.dependencyCheck = dependencyCheck;
	}

	public int getDependencyCheck() {
		return this.dependencyCheck;
	}

	public void setAutowireMode(int autowireMode) {
		this.autowireMode = autowireMode;
	}

	public int getAutowireMode() {
		return this.autowireMode;
	}

	public void setInitMethodName(@Nullable String initMethodName) {
		this.initMethodName = (StringUtils.hasText(initMethodName) ? initMethodName : null);
	}

	@Nullable
	public String getInitMethodName() {
		return this.initMethodName;
	}

	public void setDestroyMethodName(@Nullable String destroyMethodName) {
		this.destroyMethodName = (StringUtils.hasText(destroyMethodName) ? destroyMethodName : null);
	}

	@Nullable
	public String getDestroyMethodName() {
		return this.destroyMethodName;
	}

}
```

- set/get/is 方法:bean类成员相关操作

```java
/**
	 * Specify the bean class name of this bean definition.  指定此bean定义的bean类名
	 */
	@Override
	public void setBeanClassName(@Nullable String beanClassName) {
		this.beanClass = beanClassName;
	}

	/**
	 * Return the current bean class name of this bean definition. 返回此bean定义的当前bean类名
	 */
	@Override
	@Nullable
	public String getBeanClassName() {
		Object beanClassObject = this.beanClass;
		if (beanClassObject instanceof Class) { 
			return ((Class<?>) beanClassObject).getName(); //获取bean的名称
		}
		else {
			return (String) beanClassObject;
		}
	}

	/**
	 * Specify the class for this bean. 为这个bean指定类
	 */
	public void setBeanClass(@Nullable Class<?> beanClass) {
		this.beanClass = beanClass;
	}

	/**
	 * Return the class of the wrapped bean, if already resolved. 如果已经解析，返回包装bean的类
	 * @return the bean class, or {@code null} if none defined
	 * @throws IllegalStateException if the bean definition does not define a bean class,
	 * or a specified bean class name has not been resolved into an actual Class
	 */
	public Class<?> getBeanClass() throws IllegalStateException {
		Object beanClassObject = this.beanClass;
		if (beanClassObject == null) {
			throw new IllegalStateException("No bean class specified on bean definition");
		}
		if (!(beanClassObject instanceof Class)) {
			throw new IllegalStateException(
					"Bean class name [" + beanClassObject + "] has not been resolved into an actual Class");
		}
		return (Class<?>) beanClassObject;
	}

	/**
	 * Return whether this definition specifies a bean class.反回此定义是否指定bean类
	 */ 
	public boolean hasBeanClass() {
		return (this.beanClass instanceof Class);
	}

	/**
	 * Determine the class of the wrapped bean, resolving it from a 确定包装bean的类，如果需要从一个指定类解析
	 * specified class name if necessary. Will also reload a specified 还将重新加载一个根据它的名称类，
	 * Class from its name when called with the bean class already resolved.当调用时bean已经解析
	 * @param classLoader the ClassLoader to use for resolving a (potential) class name 用于解析(潜在的)类名的类加载器
	 * @return the resolved bean class
	 * @throws ClassNotFoundException if the class name could be resolved
	 */
	@Nullable
	public Class<?> resolveBeanClass(@Nullable ClassLoader classLoader) throws ClassNotFoundException {
		String className = getBeanClassName();
		if (className == null) {
			return null;
		}
		Class<?> resolvedClass = ClassUtils.forName(className, classLoader); //根据classloader加载一个类名，解析为一个类
		this.beanClass = resolvedClass;
		return resolvedClass;
	}

	/**
	 * Set the name of the target scope for the bean. 为bean设置目标范围的名称
	 * <p>The default is singleton status 默认单例, although this is only applied once 尽管这只应用一次
	 * a bean definition becomes active in the containing factory bean定义在容器工厂bean中将处理激活. A bean 
	 * definition may eventually inherit its scope from a parent bean definition. 继承他父bean定义
	 * For this reason, the default scope name is an empty string (i.e., {@code ""}), 默认为""
	 * with singleton status being assumed until a resolved scope is set. 在设置解析范围之前，假定单例状态
	 * @see #SCOPE_SINGLETON
	 * @see #SCOPE_PROTOTYPE
	 */
	@Override
	public void setScope(@Nullable String scope) {
		this.scope = scope;
	}

	/**
	 * Return the name of the target scope for the bean. 返回bean的目标范围的名称
	 */
	@Override
	@Nullable
	public String getScope() {
		return this.scope;
	}

	/**
	 * Return whether this a <b>Singleton</b>, with a single shared instance 单例，共享实例
	 * returned from all calls.
	 * @see #SCOPE_SINGLETON
	 */
	@Override
	public boolean isSingleton() {
		return SCOPE_SINGLETON.equals(this.scope) || SCOPE_DEFAULT.equals(this.scope); //singleton或者为""
	}

	/**
	 * Return whether this a <b>Prototype</b>, with an independent instance 原型，独立实例
	 * returned for each call.
	 * @see #SCOPE_PROTOTYPE
	 */
	@Override
	public boolean isPrototype() {
		return SCOPE_PROTOTYPE.equals(this.scope); //prototype
	}

	/**
	 * Set if this bean is "abstract", i.e. not meant to be instantiated itself but 如果bean是抽象设置，不打算实例化本身
	 * rather just serving as parent for concrete child bean definitions. 而是仅仅服务于具体子bean定义
	 * <p>Default is "false". Specify true to tell the bean factory to not try to 默认false, 指定true时，告诉bean工厂不尝试
	 * instantiate that particular bean in any case. 实例化
	 */
	public void setAbstract(boolean abstractFlag) {
		this.abstractFlag = abstractFlag;
	}

	/**
	 * Return whether this bean is "abstract", i.e. not meant to be instantiated
	 * itself but rather just serving as parent for concrete child bean definitions.
	 */
	@Override
	public boolean isAbstract() {
		return this.abstractFlag;
	}

	/**
	 * Set whether this bean should be lazily initialized. 置此bean是否应延迟初始化。
	 * <p>If {@code false}, the bean will get instantiated on startup by bean false时，在bean工厂执行迫切初始化单例时，进行初始化
	 * factories that perform eager initialization of singletons.
	 */
	@Override
	public void setLazyInit(boolean lazyInit) {
		this.lazyInit = lazyInit;
	}

	/**
	 * Return whether this bean should be lazily initialized, i.e. not
	 * eagerly instantiated 急切地实例化 on startup 在初始化时. Only applicable to a singleton bean. 适用于单例bean
	 */
	@Override
	public boolean isLazyInit() {
		return this.lazyInit;
	}

	/**
	 * Set the autowire mode. 自动装配模式 This determines whether any automagical detection 这决定是否有任何自动检测
	 * and setting of bean references will happen 设置bean引用. Default is AUTOWIRE_NO, 默认为AUTOWIRE_NO
	 * which means there's no autowire. 没有自动装配
	 * @param autowireMode the autowire mode to set.
	 * Must be one of the constants defined in this class.
	 * @see #AUTOWIRE_NO
	 * @see #AUTOWIRE_BY_NAME 名称
	 * @see #AUTOWIRE_BY_TYPE 类型
	 * @see #AUTOWIRE_CONSTRUCTOR 构造方法
	 * @see #AUTOWIRE_AUTODETECT 自动检测
	 */
	public void setAutowireMode(int autowireMode) {
		this.autowireMode = autowireMode;
	}

	/**
	 * Return the autowire mode as specified in the bean definition.
	 */
	public int getAutowireMode() {
		return this.autowireMode;
	}

	/**
	 * Return the resolved autowire code, 返回已解析的自动装配代码
	 * (resolving AUTOWIRE_AUTODETECT to AUTOWIRE_CONSTRUCTOR or AUTOWIRE_BY_TYPE).
	 * @see #AUTOWIRE_AUTODETECT
	 * @see #AUTOWIRE_CONSTRUCTOR
	 * @see #AUTOWIRE_BY_TYPE
	 */
	public int getResolvedAutowireMode() {
		if (this.autowireMode == AUTOWIRE_AUTODETECT) { //自动检查时
			// Work out whether to apply setter autowiring or constructor autowiring. 计算出是应用setter自动装配还是构造函数自动装配
			// If it has a no-arg constructor it's deemed to be setter autowiring, 如果它有一个无参数构造函数，它被认为是setter自动装配
			// otherwise we'll try constructor autowiring. 否则，我们将尝试构造函数自动装配
			Constructor<?>[] constructors = getBeanClass().getConstructors(); //获取类的构造函数
			for (Constructor<?> constructor : constructors) {
				if (constructor.getParameterCount() == 0) { //遍历获取参数的个数，为0时，类型装配
					return AUTOWIRE_BY_TYPE;
				}
			}
			return AUTOWIRE_CONSTRUCTOR; //构造函数装配
		}
		else {
			return this.autowireMode;
		}
	}

	/**
	 * Set the dependency check code. 设置依赖项检查
	 * @param dependencyCheck the code to set.
	 * Must be one of the four constants defined in this class. 必须是该类中定义的四个常量之一
	 * @see #DEPENDENCY_CHECK_NONE
	 * @see #DEPENDENCY_CHECK_OBJECTS
	 * @see #DEPENDENCY_CHECK_SIMPLE
	 * @see #DEPENDENCY_CHECK_ALL
	 */
	public void setDependencyCheck(int dependencyCheck) {
		this.dependencyCheck = dependencyCheck;
	}

	/**
	 * Return the dependency check code.
	 */
	public int getDependencyCheck() {
		return this.dependencyCheck;
	}

	/**
	 * Set the names of the beans that this bean depends on being initialized. 设置此bean依赖于初始化的bean的名称
	 * The bean factory will guarantee that these beans get initialized first. bean工厂将确保首先初始化这些bean
	 * <p>Note that dependencies are normally expressed through bean properties or 依赖项通常通过bean属性表示或者构造函数
	 * constructor arguments. This property should just be necessary for other kinds
	 * of dependencies like statics (*ugh*) or database preparation on startup. 此属性对于其他类型的依赖项(如启动时的静态或数据库准备)应该是必需的。
	 */
	@Override
	public void setDependsOn(@Nullable String... dependsOn) {
		this.dependsOn = dependsOn;
	}

	/**
	 * Return the bean names that this bean depends on.
	 */
	@Override
	@Nullable
	public String[] getDependsOn() {
		return this.dependsOn;
	}

	/**
	 * Set whether this bean is a candidate for getting autowired into some other bean. ：设置此bean是否为自动生成其他bean的候选bean
	 * <p>Note that this flag is designed to only affect type-based autowiring. 此标志仅用于影响基于类型的自动装配
	 * It does not affect explicit references by name, which will get resolved even 它不影响按名称显式引用，即使指定的bean没有标记为autowire候选bean，也会解析该引用
	 * if the specified bean is not marked as an autowire candidate. As a consequence, 因此
	 * autowiring by name will nevertheless inject a bean if the name matches. 如果名称匹配，按名称自动装配将注入bean
	 * @see #AUTOWIRE_BY_TYPE
	 * @see #AUTOWIRE_BY_NAME
	 */
	@Override
	public void setAutowireCandidate(boolean autowireCandidate) {
		this.autowireCandidate = autowireCandidate;
	}

	/**
	 * Return whether this bean is a candidate for getting autowired into some other bean. 
	 */
	@Override
	public boolean isAutowireCandidate() {
		return this.autowireCandidate;
	}

	/**
	 * Set whether this bean is a primary autowire candidate. 设置此bean是否是主要的自动装配候选
	 * <p>If this value is {@code true} for exactly one bean among multiple 如果此值为true ,对多个匹配候选项中的一个bean
	 * matching candidates, it will serve as a tie-breaker. 将起到决定性作用
	 */
	@Override
	public void setPrimary(boolean primary) {
		this.primary = primary;
	}

	/**
	 * Return whether this bean is a primary autowire candidate. 返回此bean是否是主要的自动装配候选
	 */
	@Override
	public boolean isPrimary() {
		return this.primary;
	}

	/**
	 * Register a qualifier to be used for autowire candidate resolution, 注册一个限定符，用于自动装配候选解析
	 * keyed by the qualifier's type name. key为限定符的类型名称
	 * @see AutowireCandidateQualifier#getTypeName()
	 */
	public void addQualifier(AutowireCandidateQualifier qualifier) {
		this.qualifiers.put(qualifier.getTypeName(), qualifier);
	}

	/**
	 * Return whether this bean has the specified qualifier. 返回此bean是否具有指定的限定符
	 */
	public boolean hasQualifier(String typeName) {
		return this.qualifiers.keySet().contains(typeName);
	}

	/**
	 * Return the qualifier mapped to the provided type name. 将限定符映射到提供的类型名称
	 */
	@Nullable
	public AutowireCandidateQualifier getQualifier(String typeName) {
		return this.qualifiers.get(typeName);
	}

	/**
	 * Return all registered qualifiers. 返回所有已注册的限定符
	 * @return the Set of {@link AutowireCandidateQualifier} objects.
	 */
	public Set<AutowireCandidateQualifier> getQualifiers() {
		return new LinkedHashSet<>(this.qualifiers.values());
	}

	/**
	 * Copy the qualifiers from the supplied AbstractBeanDefinition to this bean definition. 将限定符从提供的AbstractBeanDefinition复制到此bean定义
	 * @param source the AbstractBeanDefinition to copy from
	 */
	public void copyQualifiersFrom(AbstractBeanDefinition source) {
		Assert.notNull(source, "Source must not be null");
		this.qualifiers.putAll(source.qualifiers);
	}

	/**
	 * Specify a callback for creating an instance of the bean, 指定一个回调函数来创建bean的实例
	 * as an alternative to a declaratively specified factory method. 作为声明式指定的工厂方法的替代方法
	 * <p>If such a callback is set, it will override any other constructor
	 * or factory method metadata. However, bean property population and
	 * potential annotation-driven injection will still apply as usual.
	 * @since 5.0
	 * @see #setConstructorArgumentValues(ConstructorArgumentValues)
	 * @see #setPropertyValues(MutablePropertyValues)
	 */
	public void setInstanceSupplier(@Nullable Supplier<?> instanceSupplier) {
		this.instanceSupplier = instanceSupplier;
	}

	/**
	 * Return a callback for creating an instance of the bean, if any. 返回一个回调函数，用于创建bean的实例
	 * @since 5.0
	 */
	@Nullable
	public Supplier<?> getInstanceSupplier() {
		return this.instanceSupplier;
	}

	/**
	 * Specify whether to allow access to non-public constructors and methods, 指定是否允许访问非公共构造函数和方法，对于指向这些的外部元数据
	 * for the case of externalized metadata pointing to those. The default is 默认为true
	 * {@code true}; switch this to {@code false} for public access only. false只访问public
	 * <p>This applies to constructor resolution, factory method resolution, 这适用于构造函数解析，工厂方法解析和初始化或者销毁方法
	 * and also init/destroy methods. Bean property accessors have to be public  Bean属性访问器在任何情况下都必须是公共的，并且不受此设置的影响
	 * in any case and are not affected by this setting.
	 * <p>Note that annotation-driven configuration will still access non-public 注解驱动配置仍将访问非公共的成员
	 * members as far as they have been annotated 就它们已经注释而言. This setting applies to  此设置适用于
	 * externalized metadata in this bean definition only. 仅在此bean定义外部化元数据
	 */
	public void setNonPublicAccessAllowed(boolean nonPublicAccessAllowed) {
		this.nonPublicAccessAllowed = nonPublicAccessAllowed;
	}

	/**
	 * Return whether to allow access to non-public constructors and methods.
	 */
	public boolean isNonPublicAccessAllowed() {
		return this.nonPublicAccessAllowed;
	}

	/**
	 * Specify whether to resolve constructors in lenient mode ({@code true}, 宽松模式
	 * which is the default) or to switch to strict严格 resolution (throwing an exception
	 * in case of ambiguous constructors 模棱两可的构造函数 that all match when converting the arguments,
	 * whereas lenient mode would use the one with the 'closest' type matches). 最靠近类型匹配
	 */
	public void setLenientConstructorResolution(boolean lenientConstructorResolution) {
		this.lenientConstructorResolution = lenientConstructorResolution;
	}

	/**
	 * Return whether to resolve constructors in lenient mode or in strict mode.
	 */
	public boolean isLenientConstructorResolution() {
		return this.lenientConstructorResolution;
	}

	/**
	 * Specify the factory bean to use, if any.
	 * This the name of the bean to call the specified factory method on. 调用指定的工厂方法的bean的名称
	 * @see #setFactoryMethodName
	 */
	@Override
	public void setFactoryBeanName(@Nullable String factoryBeanName) {
		this.factoryBeanName = factoryBeanName;
	}

	/**
	 * Return the factory bean name, if any.
	 */
	@Override
	@Nullable
	public String getFactoryBeanName() {
		return this.factoryBeanName;
	}

	/**
	 * Specify a factory method, if any. This method will be invoked with 指定一个工厂方法，此方法将通过构造函数调用或者无参数
	 * constructor arguments, or with no arguments if none are specified.
	 * The method will be invoked on the specified factory bean, if any, 方法将在指定的工厂bean上调用
	 * or otherwise as a static method on the local bean class. 否则作为本地bean类上的静态方法
	 * @see #setFactoryBeanName
	 * @see #setBeanClassName
	 */
	@Override
	public void setFactoryMethodName(@Nullable String factoryMethodName) {
		this.factoryMethodName = factoryMethodName;
	}

	/**
	 * Return a factory method, if any.
	 */
	@Override
	@Nullable
	public String getFactoryMethodName() {
		return this.factoryMethodName;
	}

	/**
	 * Specify constructor argument values for this bean. 为这个bean指定构造函数参数值
	 */
	public void setConstructorArgumentValues(ConstructorArgumentValues constructorArgumentValues) {
		this.constructorArgumentValues = constructorArgumentValues;
	}

	/**
	 * Return constructor argument values for this bean (never {@code null}).
	 */
	@Override
	public ConstructorArgumentValues getConstructorArgumentValues() {
		if (this.constructorArgumentValues == null) { //为null时，创建空的ConstructorArgumentValues
			this.constructorArgumentValues = new ConstructorArgumentValues();
		}
		return this.constructorArgumentValues;
	}

	/**
	 * Return if there are constructor argument values defined for this bean.
	 */
	@Override
	public boolean hasConstructorArgumentValues() { //构造函数值不为null且不为空
		return (this.constructorArgumentValues != null && !this.constructorArgumentValues.isEmpty());
	}

	/**
	 * Specify property values for this bean, if any. 为这个bean指定属性值
	 */
	public void setPropertyValues(MutablePropertyValues propertyValues) {
		this.propertyValues = propertyValues;
	}

	/**
	 * Return property values for this bean (never {@code null}).
	 */
	@Override
	public MutablePropertyValues getPropertyValues() {
		if (this.propertyValues == null) {
			this.propertyValues = new MutablePropertyValues(); //创建空的MutablePropertyValues
		}
		return this.propertyValues;
	}

	/**
	 * Return if there are property values values defined for this bean. 如果有属性值，则返回为该bean定义的值
	 * @since 5.0.2
	 */
	@Override
	public boolean hasPropertyValues() {
		return (this.propertyValues != null && !this.propertyValues.isEmpty());
	}

	/**
	 * Specify method overrides for the bean, if any. 指定bean的方法覆盖
	 */
	public void setMethodOverrides(MethodOverrides methodOverrides) {
		this.methodOverrides = methodOverrides;
	}

	/**
	 * Return information about methods to be overridden by the IoC
	 * container. This will be empty if there are no method overrides.
	 * <p>Never returns {@code null}.
	 */
	public MethodOverrides getMethodOverrides() {
		if (this.methodOverrides == null) {
			this.methodOverrides = new MethodOverrides();
		}
		return this.methodOverrides;
	}

	/**
	 * Return if there are method overrides defined for this bean.
	 * @since 5.0.2
	 */
	public boolean hasMethodOverrides() {
		return (this.methodOverrides != null && !this.methodOverrides.isEmpty());
	}

	/**
	 * Set the name of the initializer method. 设置初始化器方法的名称
	 * <p>The default is {@code null} in which case there is no initializer method.
	 */
	public void setInitMethodName(@Nullable String initMethodName) {
		this.initMethodName = initMethodName;
	}

	/**
	 * Return the name of the initializer method.
	 */
	@Nullable
	public String getInitMethodName() {
		return this.initMethodName;
	}

	/**
	 * Specify whether or not the configured init method is the default. 指定配置的init方法是否是默认值
	 * <p>The default value is {@code false}.
	 * @see #setInitMethodName
	 */
	public void setEnforceInitMethod(boolean enforceInitMethod) {
		this.enforceInitMethod = enforceInitMethod;
	}

	/**
	 * Indicate whether the configured init method is the default. 指示所配置的init方法是否是默认值
	 * @see #getInitMethodName()
	 */
	public boolean isEnforceInitMethod() {
		return this.enforceInitMethod;
	}

	/**
	 * Set the name of the destroy method. 设置销毁方法的名称。
	 * <p>The default is {@code null} in which case there is no destroy method.
	 */
	public void setDestroyMethodName(@Nullable String destroyMethodName) {
		this.destroyMethodName = destroyMethodName;
	}

	/**
	 * Return the name of the destroy method.
	 */
	@Nullable
	public String getDestroyMethodName() {
		return this.destroyMethodName;
	}

	/**
	 * Specify whether or not the configured destroy method is the default. 指定配置的destory方法是否是默认方法
	 * <p>The default value is {@code false}.
	 * @see #setDestroyMethodName
	 */
	public void setEnforceDestroyMethod(boolean enforceDestroyMethod) {
		this.enforceDestroyMethod = enforceDestroyMethod;
	}

	/**
	 * Indicate whether the configured destroy method is the default.
	 * @see #getDestroyMethodName
	 */
	public boolean isEnforceDestroyMethod() {
		return this.enforceDestroyMethod;
	}

	/**
	 * Set whether this bean definition is 'synthetic', that is, not defined
	 * by the application itself (for example, an infrastructure bean such
	 * as a helper for auto-proxying, created through {@code <aop:config>}).
	 */
	public void setSynthetic(boolean synthetic) {
		this.synthetic = synthetic;
	}

	/**
	 * Return whether this bean definition is 'synthetic', that is,
	 * not defined by the application itself.
	 */
	public boolean isSynthetic() {
		return this.synthetic;
	}

	/**
	 * Set the role hint for this {@code BeanDefinition}.
	 */
	public void setRole(int role) {
		this.role = role;
	}

	/**
	 * Return the role hint for this {@code BeanDefinition}.
	 */
	@Override
	public int getRole() {
		return this.role;
	}


```


- set/get Description、Resource、ResourceDescription、OriginatingBeanDefinition：Bean描述、资源相关操作

```java
	/**
	 * Set a human-readable description of this bean definition. 设置bean定义的可读性描述
	 */
	public void setDescription(@Nullable String description) {
		this.description = description;
	}

	/**
	 * Return a human-readable description of this bean definition.
	 */
	@Override
	@Nullable
	public String getDescription() {
		return this.description;
	}

	/**
	 * Set the resource that this bean definition came from 设置此bean定义来自的资源
	 * (for the purpose of showing context in case of errors). 以便在出现错误时显示上下文
	 */
	public void setResource(@Nullable Resource resource) {
		this.resource = resource;
	}

	/**
	 * Return the resource that this bean definition came from.
	 */
	@Nullable
	public Resource getResource() {
		return this.resource;
	}

	/**
	 * Set a description of the resource that this bean definition 设置此bean定义来自的资源描述
	 * came from (for the purpose of showing context in case of errors).
	 */
	public void setResourceDescription(@Nullable String resourceDescription) {
		//为null时，创建描述性的资源，否则resource=null
		this.resource = (resourceDescription != null ? new DescriptiveResource(resourceDescription) : null);
	}

	/**
	 * Return a description of the resource that this bean definition
	 * came from (for the purpose of showing context in case of errors).
	 */
	@Override
	@Nullable
	public String getResourceDescription() {
		return (this.resource != null ? this.resource.getDescription() : null);
	}

	/**
	 * Set the originating (e.g. decorated 装饰) BeanDefinition, if any. 设置原始BeanDefinition
	 */
	public void setOriginatingBeanDefinition(BeanDefinition originatingBd) {
		this.resource = new BeanDefinitionResource(originatingBd); //resource引用BeanDefinitionResource对象
	}

	/**
	 * Return the originating BeanDefinition, or {@code null} if none.
	 * Allows for retrieving the decorated bean definition, if any. 允许检索修饰过的bean定义
	 * <p>Note that this method returns the immediate originator. Iterate through the 注意，此方法返回直接发起者
	 * originator chain to find the original BeanDefinition as defined by the user. 初始化链查找用户定义的原始bean
	 */
	@Override
	@Nullable
	public BeanDefinition getOriginatingBeanDefinition() {
		return (this.resource instanceof BeanDefinitionResource ? //resource为BeanDefinitionResource时，从其中直接获取BeanDefinition
				((BeanDefinitionResource) this.resource).getBeanDefinition() : null);
	}
```

- void validate()、void prepareMethodOverrides()、prepareMethodOverride(MethodOverride mo)：校验bean定义

```java
	/**
	 * Validate this bean definition. 
	 * @throws BeanDefinitionValidationException in case of validation failure 在验证失败的情况下抛出BeanDefinitionValidationException
	 */
	public void validate() throws BeanDefinitionValidationException {
		if (hasMethodOverrides() && getFactoryMethodName() != null) { //如果bean定义了重写的方法，且factoryMethodName不为null时抛出异常
			throw new BeanDefinitionValidationException(
					"Cannot combine static factory method with method overrides: " +
					"the static factory method must create the instance");
		}
		//如果存现指定bean的类
		if (hasBeanClass()) {
			prepareMethodOverrides(); //准备重写方法
		}
	}

	/**
	 * Validate and prepare the method overrides defined for this bean. 校验和准备为该bean定义的方法覆盖
	 * Checks for existence of a method with the specified name. 检查是否存在具有指定名称的方法
	 * @throws BeanDefinitionValidationException in case of validation failure
	 */
	public void prepareMethodOverrides() throws BeanDefinitionValidationException {
		// Check that lookup methods exists. 检查是否存在查找方法
		if (hasMethodOverrides()) {
			Set<MethodOverride> overrides = getMethodOverrides().getOverrides();  //遍历获取重写的方法
			synchronized (overrides) {
				for (MethodOverride mo : overrides) {
					prepareMethodOverride(mo); //准备方法重写
				}
			}
		}
	}

	/**
	 * Validate and prepare the given method override. 校验且准备给定的重写方法
	 * Checks for existence of a method with the specified name, 
	 * marking it as not overloaded if none found. 没有找到，则将其标记为未重载
	 * @param mo the MethodOverride object to validate
	 * @throws BeanDefinitionValidationException in case of validation failure
	 */
	protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
		int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName()); //根据方法名，从bean类中获取方法的个数
		if (count == 0) { //为0时抛出异常
			throw new BeanDefinitionValidationException(
					"Invalid method override: no method with name '" + mo.getMethodName() +
					"' on class [" + getBeanClassName() + "]");
		}
		else if (count == 1) { //为1时，将覆盖标记为未重载，以避免arg类型检查的开销
			// Mark override as not overloaded, to avoid the overhead of arg type checking.
			mo.setOverloaded(false);
		}
	}

```

- Object clone()、abstract AbstractBeanDefinition cloneBeanDefinition():克隆BeanDefinition

```java
/**
	 * Public declaration of Object's {@code clone()} method.
	 * Delegates to {@link #cloneBeanDefinition()}.
	 * @see Object#clone()
	 */
	@Override
	public Object clone() {
		return cloneBeanDefinition();
	}

	/**
	 * Clone this bean definition.
	 * To be implemented by concrete subclasses.  由具体子类实现
	 * @return the cloned bean definition object
	 */
	public abstract AbstractBeanDefinition cloneBeanDefinition();

```

### 总结

- bean scop:bean的作用域

	singleton
	prototype


- autowire：自动装配
	AUTOWIRE_NO=0
	AUTOWIRE_BY_NAME = 1
 	AUTOWIRE_BY_TYPE = 2
 	AUTOWIRE_CONSTRUCTOR = 3：3.x后废弃


- primary bean：多个bean候选时，注入时其主要作用
	可使用注解@Primary指定


	
## ChildBeanDefinition：用于从父Bean继承设置的Bean

继承AbstractBeanDefinition，增加parentName属性

```java
/**
 * Bean definition for beans which inherit settings from their parent.
 * Child bean definitions have a fixed dependency on a parent bean definition. 子bean定义对父bean定义具有固定的依赖关系
 *
 * <p>A child bean definition will inherit constructor argument values, 子bean定义将从父bean中继承构造参数值，属性值和重写的方法
 * property values and method overrides from the parent, with the option 具有添加新值的选项
 * to add new values. If init method, destroy method and/or static factory 如果init方法、destory方法 静态工厂方法被指定，将重写
 * method are specified, they will override the corresponding parent settings. 对应的父设置
 * The remaining settings will <i>always</i> be taken from the child definition: 剩余设置将从子bean获取
 * depends on, autowire mode, dependency check, singleton, lazy init.
 *
 * <p><b>NOTE:</b> Since Spring 2.5, the preferred way to register bean spring2.x 后，首先方式以以编程方式注册一个bean
 * definitions programmatically is the {@link GenericBeanDefinition} class,
 * which allows to dynamically define parent dependencies through the GenericBeanDefinition允许动态定义父依赖项，通过setParent方法
 * {@link GenericBeanDefinition#setParentName} method. This effectively 这有效的取代ChildBeanDefinition 类大多数情况下
 * supersedes the ChildBeanDefinition class for most use cases.

 * @see GenericBeanDefinition
 * @see RootBeanDefinition
 */
@SuppressWarnings("serial")
public class ChildBeanDefinition extends AbstractBeanDefinition {

	@Nullable
	private String parentName;


	/**
	 * Create a new ChildBeanDefinition for the given parent, to be
	 * configured through its bean properties and configuration methods.
	 * @param parentName the name of the parent bean
	 * @see #setBeanClass
	 * @see #setScope
	 * @see #setConstructorArgumentValues
	 * @see #setPropertyValues
	 */
	public ChildBeanDefinition(String parentName) {
		super();
		this.parentName = parentName;
	}

	/**
	 * Create a new ChildBeanDefinition for the given parent.
	 * @param parentName the name of the parent bean
	 * @param pvs the additional property values of the child
	 */
	public ChildBeanDefinition(String parentName, MutablePropertyValues pvs) {
		super(null, pvs);
		this.parentName = parentName;
	}

	/**
	 * Create a new ChildBeanDefinition for the given parent.
	 * @param parentName the name of the parent bean
	 * @param cargs the constructor argument values to apply
	 * @param pvs the additional property values of the child
	 */
	public ChildBeanDefinition(
			String parentName, ConstructorArgumentValues cargs, MutablePropertyValues pvs) {

		super(cargs, pvs);
		this.parentName = parentName;
	}

	/**
	 * Create a new ChildBeanDefinition for the given parent,
	 * providing constructor arguments and property values.
	 * @param parentName the name of the parent bean
	 * @param beanClass the class of the bean to instantiate
	 * @param cargs the constructor argument values to apply
	 * @param pvs the property values to apply
	 */
	public ChildBeanDefinition(
			String parentName, Class<?> beanClass, ConstructorArgumentValues cargs, MutablePropertyValues pvs) {

		super(cargs, pvs);
		this.parentName = parentName;
		setBeanClass(beanClass);
	}

	/**
	 * Create a new ChildBeanDefinition for the given parent,
	 * providing constructor arguments and property values.
	 * Takes a bean class name to avoid eager loading of the bean class.
	 * @param parentName the name of the parent bean
	 * @param beanClassName the name of the class to instantiate
	 * @param cargs the constructor argument values to apply
	 * @param pvs the property values to apply
	 */
	public ChildBeanDefinition(
			String parentName, String beanClassName, ConstructorArgumentValues cargs, MutablePropertyValues pvs) {

		super(cargs, pvs);
		this.parentName = parentName;
		setBeanClassName(beanClassName);
	}

	/**
	 * Create a new ChildBeanDefinition as deep copy of the given
	 * bean definition.
	 * @param original the original bean definition to copy from
	 */
	public ChildBeanDefinition(ChildBeanDefinition original) {
		super(original);
	}


	@Override
	public void setParentName(@Nullable String parentName) {
		this.parentName = parentName;
	}

	@Override
	@Nullable
	public String getParentName() {
		return this.parentName;
	}

	@Override
	public void validate() throws BeanDefinitionValidationException {
		super.validate();
		if (this.parentName == null) {
			throw new BeanDefinitionValidationException("'parentName' must be set in ChildBeanDefinition");
		}
	}


	@Override
	public AbstractBeanDefinition cloneBeanDefinition() {
		return new ChildBeanDefinition(this);
	}
}

```

## RootBeanDefinition：根bean定义

RootBeanDefinition继承AbstractBeanDefinition，表明它是一个可合并的bean definition：即在spring beanFactory运行期间，可以返回一个特定的bean。

RootBeanDefinition可以作为一个重要的通用的bean definition 视图，在配置文件中可以定义父和子，父用RootBeanDefinition表示， 而子用ChildBeanDefiniton表示，而没有父的就使用 RootBeanDefinition表示。

### RootBeanDefinition类描述

```java
/**
 * A root bean definition represents 表示 the merged bean definition that backs 在spring BeanFactory运行时的背后，合成特定的bean
 * a specific bean in a Spring BeanFactory at runtime. It might have been created
 * from multiple original bean definitions that inherit from each other, 它可能从相互继承的多个原始bean定义创建
 * typically registered as {@link GenericBeanDefinition GenericBeanDefinitions}. 通常作为GenericBeanDefinitions注册。在运行时
 * A root bean definition is essentially the 'unified' bean definition view at runtime. rootbean定义本质上时统一的bean定义视图
 *
 * <p>Root bean definitions may also be used for registering individual bean definitions 在配置阶段，根bean定义可能被用于
 * in the configuration phase 时期. However, since Spring 2.5, the preferred way to register 注册单个bean定义，从2.5后，
 * bean definitions programmatically is the {@link GenericBeanDefinition} class. 首先注册一个bean的方式为编程式GenericBeanDefinition
 * GenericBeanDefinition has the advantage that it allows to dynamically define GenericBeanDefinition优势在于，允许动态定义parent
 * parent dependencies, not 'hard-coding' the role as a root bean definition. 依赖，而不是将角色“硬编码”为根bean定义
 *
 * @see GenericBeanDefinition
 * @see ChildBeanDefinition
 */
@SuppressWarnings("serial")
public class RootBeanDefinition extends AbstractBeanDefinition {

}
```

### RootBeanDefinition提供的成员变量

```java

	@Nullable
	private BeanDefinitionHolder decoratedDefinition; //BeanDefinition的持有者

	@Nullable
	private AnnotatedElement qualifiedElement; //注解元素:限定符

	boolean allowCaching = true; 

	boolean isFactoryMethodUnique = false;

	@Nullable
	volatile ResolvableType targetType; //类型解析器

	/** Package-visible field for caching the determined Class of a given bean definition */ 用于缓存给定bean定义的已确定类
	@Nullable
	volatile Class<?> resolvedTargetType; 

	/** Package-visible field for caching the return type of a generically typed factory method */ 缓存泛型工厂方法的返回类型
	@Nullable
	volatile ResolvableType factoryMethodReturnType;
 
	/** Common lock for the four constructor fields below */ 下面四个构造函数字段的公共锁
	final Object constructorArgumentLock = new Object();

	/** Package-visible field for caching the resolved constructor or factory method */
	@Nullable
	Executable resolvedConstructorOrFactoryMethod; //缓存已解析的构造函数或工厂方法

	/** Package-visible field that marks the constructor arguments as resolved */
	boolean constructorArgumentsResolved = false;

	/** Package-visible field for caching fully resolved constructor arguments */ 用于缓存完全解析的构造函数参数
	@Nullable
	Object[] resolvedConstructorArguments;

	/** Package-visible field for caching partly prepared constructor arguments */ 用于缓存部分准备好的构造函数参数
	@Nullable
	Object[] preparedConstructorArguments;

	/** Common lock for the two post-processing fields below */ 下面两个后处理字段的公共锁
	final Object postProcessingLock = new Object();

	/** Package-visible field that indicates MergedBeanDefinitionPostProcessor having been applied */ 
	boolean postProcessed = false; //表明MergedBeanDefinitionPostProcessor已经应用

	/** Package-visible field that indicates a before-instantiation post-processor having kicked in */
	@Nullable
	volatile Boolean beforeInstantiationResolved; //启动的实例化前后处理

	@Nullable
	private Set<Member> externallyManagedConfigMembers;//外部管理的配置成员

	@Nullable
	private Set<String> externallyManagedInitMethods;//外部管理的初始化方法

	@Nullable
	private Set<String> externallyManagedDestroyMethods; //外部管理的销毁方法


```

### RootBeanDefinition提供的构造函数

Spring 5.x提供了Supplier<T> instanceSupplier参数 的构造函数，可以使用通过Supplier<T> get()方法或者Lambds表达式直接生产一个bean的实例

```java
	/**
	 * Create a new RootBeanDefinition, to be configured through its bean 通过beans属性和配置的方法创建一个要配置的新的RootBeanDefinition
	 * properties and configuration methods.
	 * @see #setBeanClass
	 * @see #setScope
	 * @see #setConstructorArgumentValues
	 * @see #setPropertyValues
	 */
	public RootBeanDefinition() {
		super();
	}

	/**
	 * Create a new RootBeanDefinition for a singleton. 创建一个单例的RootBeanDefinition
	 * @param beanClass the class of the bean to instantiate 要实例化的bean的类
	 * @see #setBeanClass
	 */
	public RootBeanDefinition(@Nullable Class<?> beanClass) {
		super();
		setBeanClass(beanClass);
	}

	/**
	 * Create a new RootBeanDefinition for a singleton bean, constructing each instance
	 * through calling the given supplier (possibly a lambda or method reference).  通过调用supplier构造每个实例(lambda或方法引用)
	 * @param beanClass the class of the bean to instantiate 要实例化的bean的类进入翻译页面
	 * @param instanceSupplier the supplier to construct a bean instance,
	 * as an alternative to a declaratively specified factory method 构造bean实例的supplier
	 * @since 5.0
	 * @see #setInstanceSupplier
	 */
	public <T> RootBeanDefinition(@Nullable Class<T> beanClass, @Nullable Supplier<T> instanceSupplier) {
		super();
		setBeanClass(beanClass);
		setInstanceSupplier(instanceSupplier);
	}

	/** 
	 * Create a new RootBeanDefinition for a scoped bean, constructing each instance //作用域的bean
	 * through calling the given supplier (possibly a lambda or method reference).
	 * @param beanClass the class of the bean to instantiate
	 * @param scope the name of the corresponding scope  响应作用域的名字
	 * @param instanceSupplier the supplier to construct a bean instance,
	 * as an alternative to a declaratively specified factory method 作为声明式指定的工厂方法的替代方法
	 * @since 5.0
	 * @see #setInstanceSupplier
	 */
	public <T> RootBeanDefinition(@Nullable Class<T> beanClass, String scope, @Nullable Supplier<T> instanceSupplier) {
		super();
		setBeanClass(beanClass);
		setScope(scope);
		setInstanceSupplier(instanceSupplier);
	}

	/**
	 * Create a new RootBeanDefinition for a singleton,
	 * using the given autowire mode. 使用给定的自动装配模式
	 * @param beanClass the class of the bean to instantiate
	 * @param autowireMode by name or type, using the constants in this interface 使用这个接口中的常量
	 * @param dependencyCheck whether to perform a dependency check for objects
	 * (not applicable to autowiring a constructor, thus ignored there) 不适用于自动装配构造函数，因此被忽略
	 */
	public RootBeanDefinition(@Nullable Class<?> beanClass, int autowireMode, boolean dependencyCheck) {
		super();
		setBeanClass(beanClass);
		setAutowireMode(autowireMode);
		if (dependencyCheck && getResolvedAutowireMode() != AUTOWIRE_CONSTRUCTOR) {
			setDependencyCheck(DEPENDENCY_CHECK_OBJECTS);
		}
	}

	/**
	 * Create a new RootBeanDefinition for a singleton,
	 * providing constructor arguments and property values. 提供构造函数参数和属性值
	 * @param beanClass the class of the bean to instantiate
	 * @param cargs the constructor argument values to apply
	 * @param pvs the property values to apply
	 */
	public RootBeanDefinition(@Nullable Class<?> beanClass, @Nullable ConstructorArgumentValues cargs,
			@Nullable MutablePropertyValues pvs) {

		super(cargs, pvs);
		setBeanClass(beanClass);
	}

	/**
	 * Create a new RootBeanDefinition for a singleton,
	 * providing constructor arguments and property values.
	 * <p>Takes a bean class name to avoid eager loading of the bean class.
	 * @param beanClassName the name of the class to instantiate
	 */
	public RootBeanDefinition(String beanClassName) {
		setBeanClassName(beanClassName);
	}

	/**
	 * Create a new RootBeanDefinition for a singleton,
	 * providing constructor arguments and property values.
	 * <p>Takes a bean class name to avoid eager loading of the bean class.
	 * @param beanClassName the name of the class to instantiate 提供构造函数参数和属性值
	 * @param cargs the constructor argument values to apply
	 * @param pvs the property values to apply
	 */
	public RootBeanDefinition(String beanClassName, ConstructorArgumentValues cargs, MutablePropertyValues pvs) {
		super(cargs, pvs);
		setBeanClassName(beanClassName);
	}

	/**
	 * Create a new RootBeanDefinition as deep copy of the given 深度copy给定的beandefinition
	 * bean definition.
	 * @param original the original bean definition to copy from
	 */
	public RootBeanDefinition(RootBeanDefinition original) {
		super(original);
		this.decoratedDefinition = original.decoratedDefinition;
		this.qualifiedElement = original.qualifiedElement;
		this.allowCaching = original.allowCaching;
		this.isFactoryMethodUnique = original.isFactoryMethodUnique;
		this.targetType = original.targetType;
	}

	/**
	 * Create a new RootBeanDefinition as deep copy of the given
	 * bean definition.
	 * @param original the original bean definition to copy from
	 */
	RootBeanDefinition(BeanDefinition original) {
		super(original);
	}


```

### RootBeanDefinition提供的方法

```java
	@Override
	public String getParentName() {
		return null;
	}

	@Override
	public void setParentName(@Nullable String parentName) {
		if (parentName != null) {
			throw new IllegalArgumentException("Root bean cannot be changed into a child bean with parent reference");
		}
	}

	/**
	 * Register a target definition that is being decorated by this bean definition. 注册一个通过bean定义的修饰符的目标bean定义
	 */
	public void setDecoratedDefinition(@Nullable BeanDefinitionHolder decoratedDefinition) {
		this.decoratedDefinition = decoratedDefinition;
	}

	/**
	 * Return the target definition that is being decorated by this bean definition, if any. 返回此bean定义装饰的目标定义
	 */
	@Nullable
	public BeanDefinitionHolder getDecoratedDefinition() {
		return this.decoratedDefinition;
	}

	/**
	 * Specify the {@link AnnotatedElement} defining qualifiers, 指定一个注解元素定义限定符用于代替目标class或者工厂方法
	 * to be used instead of the target class or factory method.
	 * @since 4.3.3
	 * @see #setTargetType(ResolvableType)
	 * @see #getResolvedFactoryMethod()
	 */
	public void setQualifiedElement(@Nullable AnnotatedElement qualifiedElement) {
		this.qualifiedElement = qualifiedElement;
	}

	/**
	 * Return the {@link AnnotatedElement} defining qualifiers, if any.
	 * Otherwise, the factory method and target class will be checked. 将检查工厂方法和目标类
	 * @since 4.3.3
	 */
	@Nullable
	public AnnotatedElement getQualifiedElement() {
		return this.qualifiedElement;
	}

	/**
	 * Specify a generics-containing 泛型容器 target type of this bean definition, if known in advance. 如果事先知道
	 * @since 4.3.3
	 */
	public void setTargetType(ResolvableType targetType) {
		this.targetType = targetType;
	}

	/**
	 * Specify the target type of this bean definition, if known in advance.
	 * @since 3.2.2
	 */
	public void setTargetType(@Nullable Class<?> targetType) {
		this.targetType = (targetType != null ? ResolvableType.forClass(targetType) : null);
	}

	/**
	 * Return the target type of this bean definition, if known 返回此bean定义的目标类型
	 * (either specified in advance or resolved on first instantiation 在第一次实例化时解析). 
	 * @since 3.2.2
	 */
	@Nullable
	public Class<?> getTargetType() {
		if (this.resolvedTargetType != null) {  //解析目标类型
			return this.resolvedTargetType;
		}
		ResolvableType targetType = this.targetType; //类型解析器
		return (targetType != null ? targetType.resolve() : null);
	}

	/**
	 * Specify a factory method name that refers to a non-overloaded method. 指定一个引用非重载方法的工厂方法名
	 */
	public void setUniqueFactoryMethodName(String name) {
		Assert.hasText(name, "Factory method name must not be empty");
		setFactoryMethodName(name);
		this.isFactoryMethodUnique = true;
	}

	/**
	 * Check whether the given candidate qualifies as a factory method. 检查给定的候选方法是否符合工厂方法的条件
	 */
	public boolean isFactoryMethod(Method candidate) {
		return candidate.getName().equals(getFactoryMethodName());
	}

	/**
	 * Return the resolved factory method as a Java Method object, if available.
	 * @return the factory method, or {@code null} if not found or not resolved yet
	 */
	@Nullable
	public Method getResolvedFactoryMethod() {
		synchronized (this.constructorArgumentLock) {
			Executable candidate = this.resolvedConstructorOrFactoryMethod;
			return (candidate instanceof Method ? (Method) candidate : null);
		}
	}

	public void registerExternallyManagedConfigMember(Member configMember) {
		synchronized (this.postProcessingLock) {
			if (this.externallyManagedConfigMembers == null) {
				this.externallyManagedConfigMembers = new HashSet<>(1);
			}
			this.externallyManagedConfigMembers.add(configMember);
		}
	}

	public boolean isExternallyManagedConfigMember(Member configMember) {
		synchronized (this.postProcessingLock) {
			return (this.externallyManagedConfigMembers != null &&
					this.externallyManagedConfigMembers.contains(configMember));
		}
	}

	public void registerExternallyManagedInitMethod(String initMethod) {
		synchronized (this.postProcessingLock) {
			if (this.externallyManagedInitMethods == null) {
				this.externallyManagedInitMethods = new HashSet<>(1);
			}
			this.externallyManagedInitMethods.add(initMethod);
		}
	}

	public boolean isExternallyManagedInitMethod(String initMethod) {
		synchronized (this.postProcessingLock) {
			return (this.externallyManagedInitMethods != null &&
					this.externallyManagedInitMethods.contains(initMethod));
		}
	}

	public void registerExternallyManagedDestroyMethod(String destroyMethod) {
		synchronized (this.postProcessingLock) {
			if (this.externallyManagedDestroyMethods == null) {
				this.externallyManagedDestroyMethods = new HashSet<>(1);
			}
			this.externallyManagedDestroyMethods.add(destroyMethod);
		}
	}

	public boolean isExternallyManagedDestroyMethod(String destroyMethod) {
		synchronized (this.postProcessingLock) {
			return (this.externallyManagedDestroyMethods != null &&
					this.externallyManagedDestroyMethods.contains(destroyMethod));
		}
	}


	@Override
	public RootBeanDefinition cloneBeanDefinition() { //克隆
		return new RootBeanDefinition(this);
	}

```

## GenericBeanDefinition：通用BeanDefinition

```java
/**
 * GenericBeanDefinition is a one-stop shop 一站式流水作业 for standard bean definition purposes. 用于标准bean定义目的
 * Like any bean definition, it allows for specifying a class plus optionally 像任何bean定义，GenericBeanDefinition允许
 * constructor argument values and property values. Additionally, deriving from a  可选地加上一个类的构造函数值和属性值，此外
 * parent bean definition can be flexibly configured through the "parentName" property. 从parentbean定义中得到一个能灵活地通过parent
 *name属性配置
 * <p>In general, use this {@code GenericBeanDefinition} class for the purpose of 通常，使用GenericBeanDefinition类的目的是注册一个
 * registering user-visible bean definitions (which a post-processor might operate on, 用户可见的bean定义（后处理器可能会继续工作
 * potentially even reconfiguring the parent name). Use {@code RootBeanDefinition} /  甚至可能重新配置父名称）。 
 * {@code ChildBeanDefinition} where parent/child relationships happen to be pre-determined. 关系是预先决定的
 * @see #setParentName
 * @see RootBeanDefinition
 * @see ChildBeanDefinition
 */
@SuppressWarnings("serial")
public class GenericBeanDefinition extends AbstractBeanDefinition {

	@Nullable
	private String parentName; //父bean名称


	/**
	 * Create a new GenericBeanDefinition, to be configured through its bean 通过bean的属性和配置丰富创建新的GenericBeanDefinition
	 * properties and configuration methods.
	 * @see #setBeanClass
	 * @see #setScope
	 * @see #setConstructorArgumentValues
	 * @see #setPropertyValues
	 */
	public GenericBeanDefinition() {
		super();
	}

	/**
	 * Create a new GenericBeanDefinition as deep copy of the given
	 * bean definition.
	 * @param original the original bean definition to copy from
	 */
	public GenericBeanDefinition(BeanDefinition original) {
		super(original);
	}


	@Override
	public void setParentName(@Nullable String parentName) {
		this.parentName = parentName;
	}

	@Override
	@Nullable
	public String getParentName() {
		return this.parentName;
	}


	@Override
	public AbstractBeanDefinition cloneBeanDefinition() {
		return new GenericBeanDefinition(this);
	}

}
```

## AnnotatedGenericBeanDefinition:通用注解bean定义

AnnotatedGenericBeanDefinition继承GenericBeanDefinition，并实现了AnnotatedBeanDefinition接口，内部持有AnnotationMetadata、和MethodMetadata，默认使用StandardAnnotationMetadata实现类。详见:[Spring源码-AnnotatedTypeMetadata注解类型元数据接口](/2019/05/14/spring-AnnotatedTypeMetadata/)

```java
/**
 * Extension of the {@link org.springframework.beans.factory.support.GenericBeanDefinition} 扩展GenericBeanDefinition类型
 * class, adding support for annotation metadata exposed through the 添加支持注解元数据，通过AnnotatedBeanDefinition暴露
 * {@link AnnotatedBeanDefinition} interface.
 *
 * <p>This GenericBeanDefinition variant is mainly useful for testing code that expects 多样的GenericBeanDefinition对测试代码
 * to operate on an AnnotatedBeanDefinition, for example strategy implementations 希望操作一个AnnotatedBeanDefinition非常有用，如
 * in Spring's component scanning support (where the default definition class is   spring组件扫描策略接口实现支持
 * {@link org.springframework.context.annotation.ScannedGenericBeanDefinition}, 默认的定义类是ScannedGenericBeanDefinition，也实现了
 * which also implements the AnnotatedBeanDefinition interface). AnnotatedBeanDefinition接口
 * @since 2.5
 * @see AnnotatedBeanDefinition#getMetadata()
 * @see org.springframework.core.type.StandardAnnotationMetadata
 */
@SuppressWarnings("serial")
public class AnnotatedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

	private final AnnotationMetadata metadata; //注解元数据

	@Nullable
	private MethodMetadata factoryMethodMetadata; //工厂方法元数据


	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given bean class.
	 * @param beanClass the loaded bean class
	 */
	public AnnotatedGenericBeanDefinition(Class<?> beanClass) { //为给定的bean类创建一个新的AnnotatedGenericBeanDefinition
		setBeanClass(beanClass);
		this.metadata = new StandardAnnotationMetadata(beanClass, true); //使用标准注解元数据类
	}

	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given annotation metadata, 注释元数据注释元数据
	 * allowing for ASM-based processing and avoidance of early loading of the bean class. 运行基于ASM处理和避免提前加载bean类
	 * Note that this constructor is functionally equivalent to 等价于ScannedGenericBeanDefinition
	 * {@link org.springframework.context.annotation.ScannedGenericBeanDefinition
	 * ScannedGenericBeanDefinition}, however the semantics of the latter indicate that a 然而，后者的语义表明
	 * bean was discovered specifically via component-scanning as opposed 相反 to other means. 一个bean的方法是通过指定的 有关bean类的注释元数据
	 * @param metadata the annotation metadata for the bean class in question 有关bean类的注释元数据  
	 * @since 3.1.1
	 */
	public AnnotatedGenericBeanDefinition(AnnotationMetadata metadata) {
		Assert.notNull(metadata, "AnnotationMetadata must not be null");
		if (metadata instanceof StandardAnnotationMetadata) { //标准注解元数据
			setBeanClass(((StandardAnnotationMetadata) metadata).getIntrospectedClass()); //内省beanclass
		}
		else {
			setBeanClassName(metadata.getClassName());
		}
		this.metadata = metadata;
	}

	/**
	 * Create a new AnnotatedGenericBeanDefinition for the given annotation metadata, 创建AnnotatedGenericBeanDefinition通过给定的
	 * based on an annotated class and a factory method on that class. 基于注解类的注解元数据和类上的工厂方法
	 * @param metadata the annotation metadata for the bean class in question
	 * @param factoryMethodMetadata metadata for the selected factory method 所选工厂方法的元数据
	 * @since 4.1.1
	 */
	public AnnotatedGenericBeanDefinition(AnnotationMetadata metadata, MethodMetadata factoryMethodMetadata) {
		this(metadata);
		Assert.notNull(factoryMethodMetadata, "MethodMetadata must not be null");
		setFactoryMethodName(factoryMethodMetadata.getMethodName());
		this.factoryMethodMetadata = factoryMethodMetadata;
	}


	@Override
	public final AnnotationMetadata getMetadata() {
		 return this.metadata;
	}

	@Override
	@Nullable
	public final MethodMetadata getFactoryMethodMetadata() {
		return this.factoryMethodMetadata;
	}

}

```

## ScannedGenericBeanDefinition：扫描Bean定义

ScannedGenericBeanDefinition基于ASM ClassReader读取类中的信息，而不需要加载当前类，以提升类解析性能

```java
/**
 * Extension of the {@link org.springframework.beans.factory.support.GenericBeanDefinition} 扩展GenericBeanDefinition类，基于
 * class, based on an ASM ClassReader, with support for annotation metadata exposed ASM ClassReader类读取器，通过AnnotatedBeanDefinition接口，支持暴露ASM ClassReader类读取器
 * through the {@link AnnotatedBeanDefinition} interface.
 *
 * <p>This class does <i>not</i> load the bean {@code Class} early. Bean类不过早的加载，而是从class文件中检索所有相关的元数据
 * It rather retrieves all relevant metadata from the ".class" file itself, 
 * parsed with the ASM ClassReader 使用ASM类读取器解析. It is functionally equivalent to 等价于
 * {@link AnnotatedGenericBeanDefinition#AnnotatedGenericBeanDefinition(AnnotationMetadata)} AnnotatedGenericBeanDefinition(AnnotationMetadata metadata)，但是区别在于通过已经扫描的类型bean 对比那些另外的已经注册或检索的bean(通过其他方式)
 * but distinguishes by type beans that have been <em>scanned</em> vs those that have
 * been otherwise registered 注册 or detected 检测by other means.
 * @since 2.5
 * @see #getMetadata()
 * @see #getBeanClassName()
 * @see org.springframework.core.type.classreading.MetadataReaderFactory 元数据读取工厂
 * @see AnnotatedGenericBeanDefinition
 */
@SuppressWarnings("serial")
public class ScannedGenericBeanDefinition extends GenericBeanDefinition implements AnnotatedBeanDefinition {

	private final AnnotationMetadata metadata;


	/**
	 * Create a new ScannedGenericBeanDefinition for the class that the 通过给定类的MetadataReader描述创建ScannedGenericBeanDefinition
	 * given MetadataReader describes.
	 * @param metadataReader the MetadataReader for the scanned target class 扫描目标类的元数据阅读器
	 */
	public ScannedGenericBeanDefinition(MetadataReader metadataReader) {
		Assert.notNull(metadataReader, "MetadataReader must not be null");
		this.metadata = metadataReader.getAnnotationMetadata();
		setBeanClassName(this.metadata.getClassName());
	}


	@Override
	public final AnnotationMetadata getMetadata() {
		return this.metadata;
	}

	@Override
	@Nullable
	public MethodMetadata getFactoryMethodMetadata() {
		return null;
	}

}

```

