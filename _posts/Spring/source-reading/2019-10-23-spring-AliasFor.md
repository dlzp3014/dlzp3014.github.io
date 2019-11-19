---
layout: post
title:  "Spring源码-@AliasFor别名注解"
date:   2019-10-23 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Annotation
---

* content
{:toc}

@AliasFor是一个用于声明注释属性别名的注释。API说明如下：



## 使用场景:

- 注解内显式别名: 在单个注解内，@AliasFor可以将一对属性声明为可交换的别名(`interchangeable aliases`)

- 元注解中对属性显式别名: 如果@AliasFor的`annotation`属性设置了不同的注解，而不是声明它的(如@Service中声明的value属性`@AliasFor(annotation = Component.class)`，指定了@Componenet注解)，`attribute`属性值被解释为元注解内的属性别名(显式元注释属性覆盖)，这使得细粒度(fine-grained)的准确控制在注解属性层级中要覆盖的属性。实际上，使用@AliasFor甚至有可能对一个元注解的`value`属性声明一个别名

- 注解中隐式别名: 如果注解中的一个或多个属性被声明为同一元注解属性的属性覆盖(直接的或传递的)，这些属性将被视为彼此的一组隐式别名，结果导致类似于(analogous)注释中显式别名的行为


## 使用要求

与Java中的任何注释一样，仅@AliasFor本身的存在不会强制别名的语义，为了强制别名的语义，必须通过[AnnotationUtils:注解公共方法加载注解](/2019/10/22/spring-AnnotationUtils/)。在本质上，Spring将合成(`synthesize`)一个注解，通过对使用@AliasFor注解注释的注解属性将其包装在一个显式的强制属性别名语义的动态代理中，类似的，当AliasFor用于注解层级时，[AnnotatedElementUtils:注解元素工具类](/2019/10/21/spring-AnnotatedElementUtils/)支持显式元注释属性覆盖。通常情况下，不需要手动`manually`合成注解，因为当在Spring管理的组件上查找注解时，Spring会透明的进行处理


## 实现要求

- 注解内显式别名

1.组成别名对的每个属性必须使用@AliasFor注释，并且`attribute() or value()` 必须引用别名对中的其他属性

2.别名属性必须声明相同的返回值

3.别名属性必须声明一个默认值

4.别名属性必须声明相同的默认值

5.`annotation()`不需要声明

- 元注解中对属性显式别名

1.元注释中属性的别名必须使用` @AliasFor`注释，且`attribute()`必须引用元注解中的属性

2.别名属性必须声明相同的返回类型

3.`attribute()`必须引用元注解

4.引用元数据必须是声明@AliasFor注解类存在的元数据


- 注解中隐式别名

1.属于隐式集合中的每一个别名必须使用@AliasFor注释，且` attribute()`必须引用在同一元注解相同的属性(直接或间接地通过其他显式元注释属性覆盖注释层次结构)

2.别名属性必须声明相同的返回值

3.别名属性必须声明一个默认值

4.别名属性必须声明相同的默认值

5.` attribute()`必须引用一个适当的元注解

6.被引用的元注释必须是存在于声明@AliasFor的注释类

## 示例

- 显式注解别名

注解中的属性别名对

```java
public @interface ContextConfiguration {

    @AliasFor("locations")
    String[] value() default {};

    @AliasFor("value")
    String[] locations() default {};

    // ...
 }
```

- 显式元注解别名

```java
 @ContextConfiguration
 public @interface XmlTestConfig {
 	//xmlFile是@ContextConfiguration中locations的显式别名，即xmlFiles 覆盖@ContextConfiguration中的locations 属性
    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] xmlFiles();
 
```

- 隐式注解别名

In @MyTestConfig, value, groovyScripts, and xmlFiles are all explicit meta-annotation attribute overrides for the locations attribute in @ContextConfiguration`显式元注解属性覆盖@ContextConfiguration中的locations属性`. These three attributes are therefore also implicit aliases for each other(因此，这三个属性(value、groovyScripts、xmlFiles)也是彼此的隐式别名。).


```java
@ContextConfiguration
 public @interface MyTestConfig {

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] value() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] groovyScripts() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] xmlFiles() default {};
 }

```

- 传递隐式注解别名

In @GroovyOrXmlTestConfig, groovy is an explicit override for the groovyScripts attribute in @MyTestConfig; whereas, xml is an explicit override for the locations attribute in @ContextConfiguration. Furthermore, groovy and xml are transitive implicit aliases for each other(彼此的传递隐式别名), since they both effectively override the locations attribute in @ContextConfiguration.`因为它们都有效地覆盖了@ContextConfiguration中的locations属性`

```java
 @MyTestConfig
 public @interface GroovyOrXmlTestConfig {

    @AliasFor(annotation = MyTestConfig.class, attribute = "groovyScripts")
    String[] groovy() default {};

    @AliasFor(annotation = ContextConfiguration.class, attribute = "locations")
    String[] xml() default {};
 }
```

## @AliasFor注解定义

```java
/* 
 * @since 4.2
 * @see AnnotatedElementUtils
 * @see AnnotationUtils
 * @see AnnotationUtils#synthesizeAnnotation(Annotation, java.lang.reflect.AnnotatedElement)
 * @see SynthesizedAnnotation
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Documented
public @interface AliasFor {

	/** 当没有声明` #annotation`时，代替#attribute
	 * Alias for {@link #attribute}.
	 * <p>Intended to be used instead of {@link #attribute} when {@link #annotation}
	 * is not declared &mdash; for example: {@code @AliasFor("value")} instead of
	 * {@code @AliasFor(attribute = "value")}.
	 */
	@AliasFor("attribute")
	String value() default "";

	/**
	 * The name of the attribute that <em>this</em> attribute is an alias for. 属性是的别名
	 * @see #value
	 */
	@AliasFor("value")
	String attribute() default "";

	/** #attribute声明的注解类型命名
	 * The type of annotation in which the aliased {@link #attribute} is declared.
	 * <p>Defaults to {@link Annotation}, implying 暗示 that the aliased attribute is
	 * declared in the same annotation as <em>this</em> attribute.  这意味着别名属性是在与此属性相同的注释中声明的
	 */
	Class<? extends Annotation> annotation() default Annotation.class;

}


```
