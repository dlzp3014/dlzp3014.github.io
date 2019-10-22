---
layout: post
title:  "Spring源码-MethodMetadata方法元数据接口"
date:   2019-10-19 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

MethodMetadata方法元数据接口定义了访问`特定类的注解，而不需要加载当前类`，由Spring3.0版本提供，其继承了AnnotatedTypeMetadata注解类型源数据接口，详情：[Spring源码-AnnotatedTypeMetadata注解类型元数据接口](/2019/10/19/spring-AnnotatedTypeMetadata/)，表明其具备了访问方法上的注解能力。Spring框架提供的实现有StandardMethodMetadata标准方法元数据实现类(基于反射)和MethodMetadataReadingVisitor方法元数据读取访问器(基于ASM技术)。类继承关系如下：

![](/img/post.img/spring/MethodMetadata.png)





## MethodMetadata：方法元数据

```java
/**
 * Interface that defines abstract access to the annotations of a specific
 * class, in a form that does not require that class to be loaded yet.
 *
 * @since 3.0
 * @see StandardMethodMetadata
 * @see AnnotationMetadata#getAnnotatedMethods 获取类中对应注解的方法
 * @see AnnotatedTypeMetadata
 */
public interface MethodMetadata extends AnnotatedTypeMetadata {

	/**
	 * Return the name of the method.方法名
	 */
	String getMethodName();

	/** 返回声明此方法的类完全限定名
	 * Return the fully-qualified name of the class that declares this method.
	 */
	String getDeclaringClassName();

	/** 方法返回值类型
	 * Return the fully-qualified name of this method's declared return type.
	 * @since 4.2
	 */
	String getReturnTypeName();

	/**
	 * Return whether the underlying method is effectively abstract:
	 * i.e. marked as abstract on a class or declared as a regular,
	 * non-default method in an interface. 接口中的非默认方法
	 * @since 4.2
	 */
	boolean isAbstract();

	/**
	 * Return whether the underlying method is declared as 'static'.
	 */
	boolean isStatic();

	/**
	 * Return whether the underlying method is marked as 'final'.
	 */
	boolean isFinal();

	/**
	 * Return whether the underlying method is overridable,
	 * i.e. not marked as static, final or private. 没有标记为静态的、最终的或私有的
	 */
	boolean isOverridable();

}

```

## StandardMethodMetadata:标准方法元数据


StandardMethodMetadata中关于对AnnotatedTypeMetadata接口中定义的方法实现都使用了AnnotatedElementUtils注解元素工具类，其内部实现过程请查看：[Spring源码-AnnotatedElementUtils注解元素工具类](/2019/10/21/spring-AnnotatedElementUtils/)

```java 
/**
 * {@link MethodMetadata} implementation that uses standard reflection
 * to introspect a given {@code Method}. 标准反射内省给定的方法
 *
 * @since 3.0
 */
public class StandardMethodMetadata implements MethodMetadata {

	private final Method introspectedMethod;

	private final boolean nestedAnnotationsAsMap;


	/**
	 * Create a new StandardMethodMetadata wrapper包装 for the given Method.
	 * @param introspectedMethod the Method to introspect
	 */
	public StandardMethodMetadata(Method introspectedMethod) {
		this(introspectedMethod, false);
	}

	/**
	 * Create a new StandardMethodMetadata wrapper for the given Method,
	 * providing the option to return any nested annotations 嵌套注释 or annotation arrays数组 in the
	 * form of {@link org.springframework.core.annotation.AnnotationAttributes} instead
	 * of actual {@link java.lang.annotation.Annotation} instances.
	 * @param introspectedMethod the Method to introspect
	 * @param nestedAnnotationsAsMap return nested annotations and annotation arrays as
	 * {@link org.springframework.core.annotation.AnnotationAttributes} for compatibility 兼容
	 * with ASM-based {@link AnnotationMetadata} implementations
	 * @since 3.1.1
	 */
	public StandardMethodMetadata(Method introspectedMethod, boolean nestedAnnotationsAsMap) {
		Assert.notNull(introspectedMethod, "Method must not be null");
		this.introspectedMethod = introspectedMethod;
		this.nestedAnnotationsAsMap = nestedAnnotationsAsMap;
	}


	/**
	 * Return the underlying Method.
	 */
	public final Method getIntrospectedMethod() {
		return this.introspectedMethod;
	}

	@Override
	public String getMethodName() {
		return this.introspectedMethod.getName();
	}

	@Override
	public String getDeclaringClassName() {
		return this.introspectedMethod.getDeclaringClass().getName();
	}

	@Override
	public String getReturnTypeName() {
		return this.introspectedMethod.getReturnType().getName();
	}

	@Override
	public boolean isAbstract() {
		return Modifier.isAbstract(this.introspectedMethod.getModifiers());
	}

	@Override
	public boolean isStatic() {
		return Modifier.isStatic(this.introspectedMethod.getModifiers());
	}

	@Override
	public boolean isFinal() {
		return Modifier.isFinal(this.introspectedMethod.getModifiers());
	}

	@Override
	public boolean isOverridable() {
		return (!isStatic() && !isFinal() && !Modifier.isPrivate(this.introspectedMethod.getModifiers()));
	}

	@Override
	public boolean isAnnotated(String annotationName) {
		return AnnotatedElementUtils.isAnnotated(this.introspectedMethod, annotationName);
	}

	@Override
	@Nullable
	public Map<String, Object> getAnnotationAttributes(String annotationName) {
		return getAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public Map<String, Object> getAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		return AnnotatedElementUtils.getMergedAnnotationAttributes(this.introspectedMethod,
				annotationName, classValuesAsString, this.nestedAnnotationsAsMap);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName) {
		return getAllAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		return AnnotatedElementUtils.getAllAnnotationAttributes(this.introspectedMethod,
				annotationName, classValuesAsString, this.nestedAnnotationsAsMap);
	}

}
```


## MethodMetadataReadingVisitor：方法元数据读取访问器

StandardMethodMetadata中关于对AnnotatedTypeMetadata接口中定义的方法实现都使用了AnnotationReadingVisitorUtils注解元素工具类，其内部实现过程请查看：[Spring源码-AnnotationReadingVisitorUtils基于ASM注解读取访问器工具类](/2019/10/21/spring-AnnotationReadingVisitorUtils/)

```java
/**
 * ASM method visitor which looks for the annotations defined on a method,
 * exposing them through the {@link org.springframework.core.type.MethodMetadata}
 * interface.
 *
 * @since 3.0
 */
public class MethodMetadataReadingVisitor extends MethodVisitor implements MethodMetadata {

	protected final String methodName;

	protected final int access; 访问修饰符

	protected final String declaringClassName;

	protected final String returnTypeName;

	@Nullable
	protected final ClassLoader classLoader;

	protected final Set<MethodMetadata> methodMetadataSet; 方法元数据集合

	protected final Map<String, Set<String>> metaAnnotationMap = new LinkedHashMap<>(4);

	//方法上注解属性集合
	protected final LinkedMultiValueMap<String, AnnotationAttributes> attributesMap = new LinkedMultiValueMap<>(4);


	public MethodMetadataReadingVisitor(String methodName, int access, String declaringClassName,
			String returnTypeName, @Nullable ClassLoader classLoader, Set<MethodMetadata> methodMetadataSet) {

		super(SpringAsmInfo.ASM_VERSION);
		this.methodName = methodName;
		this.access = access;
		this.declaringClassName = declaringClassName;
		this.returnTypeName = returnTypeName;
		this.classLoader = classLoader;
		this.methodMetadataSet = methodMetadataSet;
	}

	注解方法器

	@Override
	public AnnotationVisitor visitAnnotation(final String desc, boolean visible) {
		this.methodMetadataSet.add(this);
		String className = Type.getType(desc).getClassName();
		return new AnnotationAttributesReadingVisitor(
				className, this.attributesMap, this.metaAnnotationMap, this.classLoader);
	}


	@Override
	public String getMethodName() {
		return this.methodName;
	}

	@Override
	public boolean isAbstract() {
		return ((this.access & Opcodes.ACC_ABSTRACT) != 0);
	}

	@Override
	public boolean isStatic() {
		return ((this.access & Opcodes.ACC_STATIC) != 0);
	}

	@Override
	public boolean isFinal() {
		return ((this.access & Opcodes.ACC_FINAL) != 0);
	}

	@Override
	public boolean isOverridable() {
		return (!isStatic() && !isFinal() && ((this.access & Opcodes.ACC_PRIVATE) == 0));
	}

	@Override
	public boolean isAnnotated(String annotationName) {
		return this.attributesMap.containsKey(annotationName);
	}

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
				"method '" + getMethodName() + "'", this.classLoader, raw, classValuesAsString);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName) {
		return getAllAnnotationAttributes(annotationName, false);
	}

	@Override
	@Nullable
	public MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName, boolean classValuesAsString) {
		if (!this.attributesMap.containsKey(annotationName)) {
			return null;
		}
		MultiValueMap<String, Object> allAttributes = new LinkedMultiValueMap<>();
		List<AnnotationAttributes> attributesList = this.attributesMap.get(annotationName);
		if (attributesList != null) {
			for (AnnotationAttributes annotationAttributes : attributesList) {
				AnnotationAttributes convertedAttributes = AnnotationReadingVisitorUtils.convertClassValues(
						"method '" + getMethodName() + "'", this.classLoader, annotationAttributes, classValuesAsString);
				convertedAttributes.forEach(allAttributes::add); //添加所有
			}
		}
		return allAttributes;
	}

	@Override
	public String getDeclaringClassName() {
		return this.declaringClassName;
	}

	@Override
	public String getReturnTypeName() {
		return this.returnTypeName;
	}

}
```



