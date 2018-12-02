---
layout: post
title:  "JAVA中通过Hibernate-Validation进行参数验证"
date:   2018-12-01 14:02:00
categories: Java
tags: Java Validator
---

* content
{:toc}

JAVA中对外部传来的参数合法性进行验证时可以使用javax.validation规范来进行，具体实现为hibernate-validator。[官网](http://hibernate.org/validator/)，使用方式如下



## Maven中添加依赖

```java
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.12.Final</version>
</dependency>
<dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.el</artifactId>
    <version>3.0.1-b10</version>
</dependency>

```

## 获取Validator实例

- 使用DefaultValidatorFactory默认校验工厂

```java
private static Validator validator = Validation.buildDefaultValidatorFactory().getValidator();
```

- 使用Validation.byProvider指定具体实现

```java
private static Validator validator = Validation.byProvider(HibernateValidator.class)
            .configure()
            .buildValidatorFactory()
            .getValidator();
```
`【note】:`由于Validation支持在一个应用使用多个提供商进行工作，如果超过一个供应商在类路径下，那么通过buildDefaultValidatorFactory()创建factory是无法保证所使用的是哪个供应商。在这种情况下，可以通过Validation.byProvider()来明确指定使用的是哪个供应商。每个供应商都会jar包含一个这样的文件：`META-INF/services/javax.validation.spi.ValidationProvider，内容为org.hibernate.validator.HibernateValidator`

- Validator接口说明

Validator接口中包含三个校验方法，都返回一个`Set<ConstraintViolation>`集合，当验证通过时当前集合为空；没有验证通过时，则会分别为没有验证通过的约束创建ConstraintViolation实例，并添加到Set<ConstraintViolation>集合中，接口定义如下：
```java
/**
 * Validates all constraints on {@code object}.
 *
 * @param object object to validate
 * @param groups the group or list of groups targeted for validation (defaults to
 *        {@link Default})
 * @param <T> the type of the object to validate
 * @return constraint violations or an empty set if none
 * @throws IllegalArgumentException if object is {@code null}
 *         or if {@code null} is passed to the varargs groups
 * @throws ValidationException if a non recoverable error happens
 *         during the validation process
 */
<T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups); //对实体进行验证 

/**
 * Validates all constraints placed on the property of {@code object}
 * named {@code propertyName}.
 *
 * @param object object to validate
 * @param propertyName property to validate (i.e. field and getter constraints)
 * @param groups the group or list of groups targeted for validation (defaults to
 *        {@link Default})
 * @param <T> the type of the object to validate
 * @return constraint violations or an empty set if none
 * @throws IllegalArgumentException if {@code object} is {@code null},
 *         if {@code propertyName} is {@code null}, empty or not a valid object property
 *         or if {@code null} is passed to the varargs groups
 * @throws ValidationException if a non recoverable error happens
 *         during the validation process
 */
<T> Set<ConstraintViolation<T>> validateProperty(T object,
                                                 String propertyName,
                                                 Class<?>... groups); //对实体中某个属性进行验证 

/**
 * Validates all constraints placed on the property named {@code propertyName}
 * of the class {@code beanType} would the property value be {@code value}.
 * <p>
 * {@link ConstraintViolation} objects return {@code null} for
 * {@link ConstraintViolation#getRootBean()} and
 * {@link ConstraintViolation#getLeafBean()}.
 *
 * @param beanType the bean type
 * @param propertyName property to validate
 * @param value property value to validate
 * @param groups the group or list of groups targeted for validation (defaults to
 *        {@link Default}).
 * @param <T> the type of the object to validate
 * @return constraint violations or an empty set if none
 * @throws IllegalArgumentException if {@code beanType} is {@code null},
 *         if {@code propertyName} is {@code null}, empty or not a valid object property
 *         or if {@code null} is passed to the varargs groups
 * @throws ValidationException if a non recoverable error happens
 *         during the validation process
 */
<T> Set<ConstraintViolation<T>> validateValue(Class<T> beanType,
                                              String propertyName,
                                              Object value,
                                              Class<?>... groups); //对实体中某个属性的值进行验证
```

## Validator Annotation

- 常用注解

@NotNull | 引用类型 | 注解元素必须非空
@Null | 引用类型 |元素为空
@Digits | byte,short,int,long及其包装器,BigDecimal,BigInteger,String| 验证数字是否合法。属性：integer(整数部分), fraction(小数部分)
@Future/@Past| java.util.Date, java.util.Calendar | 是否在当前时间之后或之前
@Max/@Min | byte,short,int,long及其包装器,BigDecimal,BigInteger | 验证值是否小于等于最大指定整数值或大于等于最小指定整数值
@Pattern | String |验证字符串是否匹配指定的正则表达式。属性：regexp(正则), flags（选项,Pattern.Flag值）
@Size | String, Collection, Map， 数组 | 验证元素大小是否在指定范围内。属性:max(最大长度), min(最小长度), message(提示，默认为{constraint.size})
@DecimalMax/@DecimalMin | byte,short,int,long及其包装器,BigDecimal,BigInteger,String | 验证值是否小于等于最大指定小数值或大于等于最小指定小数值
@Valid | |验证值是否需要递归调用

- Hibernate Validator

@Email 被注释的元素必须是电子邮箱地址
@Length 字符串的大小必须在指定的范围内
@NotEmpty 被注释的字符串的必须非空
@Range 被注释的元素必须在合适的范围内

## 自定义Validator Annotation

```java
/**
 * @Description: 自定义金额校验注解
 * @Author: dlzp
 * @Date: 2018/12/2 0002 17:44
 */
@Target({ElementType.FIELD, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy=MoneyValidator.class) //校验器的具体实现
public @interface  MoneyValidatorAnnotation {
    String message() default"金额格式不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

/**
 * @Description: 金额校验器
 * @Author: dlzp
 * @Date: 2018/12/2 0002 17:45
 */
public class MoneyValidator implements ConstraintValidator<MoneyValidatorAnnotation,Double> {
    private String moneyReg = "^\\d+(\\.\\d{1,2})?$";//表示金额的正则表达式
    private Pattern moneyPattern = Pattern.compile(moneyReg);
    @Override
    public void initialize(MoneyValidatorAnnotation constraintAnnotation) {
    }

    @Override
    public boolean isValid(Double value, ConstraintValidatorContext context) {
        return moneyPattern.matcher(value.toString()).matches();
    }
}
```

## 验证工具类

```java
/**
 * @Description: hibernate-validator工具类
 * @Author: dlzp
 * @Date: 2018/12/2 0002 15:29
 */
public class ValidationUtils {

    private static Validator validator = Validation.byProvider(HibernateValidator.class)
            .configure()
            .failFast(true)//快速失败原则：只有一个属性验证失败后即返回验证失败信息，其他属性不在校验
            .buildValidatorFactory()
            .getValidator();

    public static <T> void validate(T t, Class<?>... groups) throws ValidationException{
        Set<ConstraintViolation<T>> constraintViolations = validator.validate(t);
        constraintViolations.forEach(b -> {
            throw new ValidationException(b.getMessage());
        });
    }

    @Test
    public void validatorTest(){
        try {
            validate(new Person());
        } catch (ValidationException e) {
            Assert.assertEquals("名称不为空",e.getMessage());
        }
    }
}
```
