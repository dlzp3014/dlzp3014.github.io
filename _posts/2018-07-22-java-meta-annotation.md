---
layout: post
title:  "Java元注解"
date:   2018-07-21 15:57:23
categories: Java 
tags: Java-Annotation
---

* content
{:toc}

元注解专职负责注解其他的注解，用来标示其他注解的作用范围和作用域，大多数情况下，只需要定义自己的注解，并编写自己的处理利器来处理，具体使用过程请查询[Java Annotation 注解](/2018/07/22/java-annotation/)。其中包括@Retention @Target @Document @Inherited四种：



## @Target

表示该注解可以用于什么地方。可能的ElementType参数包括：

- CONSTRUCTOR：构造器的生命

- FIELD：域声明（包括enum实例）

- LOCAL_VARIABLE：局部变量声明

- METHOD：方法声明

- PACKAGE：包声明

- PARAMETER：参数声明

- TYPE：类、接口（包括注解类型）和enum声明

- Java1.8新增注解
ANNOTATION_TYPE、TYPE_PARAMETER、TYPE_USE   

@Target参数具体定义在ElementType枚举类中：

```java
/**
 * The constants of this enumerated type provide a simple classification of the
 * syntactic locations where annotations may appear in a Java program. These
 * constants are used in {@link Target java.lang.annotation.Target}
 * meta-annotations to specify where it is legal to write annotations of a
 * given type.
 *
 * <p>The syntactic locations where annotations may appear are split into
 * <em>declaration contexts</em> , where annotations apply to declarations, and
 * <em>type contexts</em> , where annotations apply to types used in
 * declarations and expressions.
 *
 * <p>The constants {@link #ANNOTATION_TYPE} , {@link #CONSTRUCTOR} , {@link
 * #FIELD} , {@link #LOCAL_VARIABLE} , {@link #METHOD} , {@link #PACKAGE} ,
 * {@link #PARAMETER} , {@link #TYPE} , and {@link #TYPE_PARAMETER} correspond
 * to the declaration contexts in JLS 9.6.4.1.
 *
 * <p>For example, an annotation whose type is meta-annotated with
 * {@code @Target(ElementType.FIELD)} may only be written as a modifier for a
 * field declaration.
 *
 * <p>The constant {@link #TYPE_USE} corresponds to the 15 type contexts in JLS
 * 4.11, as well as to two declaration contexts: type declarations (including
 * annotation type declarations) and type parameter declarations.
 *
 * <p>For example, an annotation whose type is meta-annotated with
 * {@code @Target(ElementType.TYPE_USE)} may be written on the type of a field
 * (or within the type of the field, if it is a nested, parameterized, or array
 * type), and may also appear as a modifier for, say, a class declaration.
 *
 * <p>The {@code TYPE_USE} constant includes type declarations and type
 * parameter declarations as a convenience for designers of type checkers which
 * give semantics to annotation types. For example, if the annotation type
 * {@code NonNull} is meta-annotated with
 * {@code @Target(ElementType.TYPE_USE)}, then {@code @NonNull}
 * {@code class C {...}} could be treated by a type checker as indicating that
 * all variables of class {@code C} are non-null, while still allowing
 * variables of other classes to be non-null or not non-null based on whether
 * {@code @NonNull} appears at the variable's declaration.
 *
 * @author  Joshua Bloch
 * @since 1.5
 * @jls 9.6.4.1 @Target
 * @jls 4.1 The Kinds of Types and Values
 */
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Formal parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE,

    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER,

    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE
}
```

## @Retention

表示需要在什么级别保存该注解信息。可选的RetentionPolicy参数包括：

- SOURCE：注解将在编译器丢弃，即仅存在于源码中，在class字节码文件中不包含

- CLASS：注解在class文件中可用，但运行时无法获得，即会被VM丢弃

- RUNTIME：VM将在运行期也保留注解，因此可以通过反射机制读取注解的信息

使用场景：如果需要在运行时去动态获取注解信息，那只能用 RUNTIME 注解；如果要在编译时进行一些预处理操作，比如生成一些辅助代码就用CLASS注解；如果只是做一些检查性的操作，比如 @Override 和 @SuppressWarnings，则可选用 SOURCE 注解。

RetentionPolicy定义如下

```java
/**
 * Annotation retention policy.  The constants of this enumerated type
 * describe the various policies for retaining annotations.  They are used
 * in conjunction with the {@link Retention} meta-annotation type to specify
 * how long annotations are to be retained.
 *
 * @author  Joshua Bloch
 * @since 1.5
 */
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE,

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME
}
```

## @Documented
将此注解包含在Javadoc中

## @Inherited
允许子类继承父类中的注解
   

