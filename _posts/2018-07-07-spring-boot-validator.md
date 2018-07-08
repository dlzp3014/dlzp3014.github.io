---
layout: post
title:  "Spring Boot 校验框架"
date:   2018-07-06 17:33:54
categories: SpringBoot
tags: SpringBoot Validator
---

* content
{:toc}

Spring Boot支持JSR3-03验证框架，默认采用的是[Hibernate validator](http://hibernate.org/validator/documentation/),[中文文档](http://docs.jboss.org/hibernate/validator/5.1/reference/zh-CN/)。`spring-boot-starter-web包里面有hibernate-validator包，不需要引用hibernate validator依赖`。在Spring MVC中，只需要使用@Valid注解标注在方法参数上，Spring Boot即可对参数进行校验，校验结果放在`BindingResult`对象中。





## JSR-303
JSR-303是Java标准的校验框架，已有的实现有Hibernate validator。JSR-303定义了一些列注解用例验证Bean的属性，常用的有如下几种：
	
### 空检查
@Null 验证对象为空  
@NotNull 验证对象不为空  
@NotBlank 验证字符串不为空或者不是空字符串，`""`和`" "`都会验证失败  
@NotEmpty 验证对象不为null或者集合不为空  

### 长度检查  
@Size(min=,max=) 验证对象长度，支持`字符串和集合`  
@Length 字符串长度  
	
### 数值检查
@Min 验证数字是否大于等于指定的值  
@Max 验证数字是小于等于指定的值
	
@DecimalMax(value)  
@DecimalMin(value)  

@Digits 验证数字是否符合指定格式，eg @Digits(integer=9,fraction=2) `限制必须为一个小数，整数部分的位数不能超过integer，小数部分的位数不能超过fraction`  
@Range 验证数字是否在指定的范围内，eg @Range(min=1,max=24)  
	
###	其他
@Email 验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式 ，为null则不做校验  
@Pattern(value) 验证String对象是否符合正则表达式的规则，eg @Pattern(regexp="^[a-zA-Z0-9]+$",message="{account.username.space}")  
    
@AssertFalse 限制必须为false  
@AssertTrue 限制必须为true  
	
@Future	验证注解的元素值是一个将来的日期  
@Past 验证注解的元素值（日期类型）比当前时间早  
	
### Example
```java
public class UserEntity {
    @NotNull
    private Long id;

    @Size(min = 3, max = 20)
    private String userName;

}
```

## 校验模式
Hibernate Validator有以下两种验证模式：一次性返回了所有验证不通过的集合；按顺序验证到第一个字段不符合验证要求时，就可以直接拒绝请求了

### 普通模式（default）
普通模式(会校验完所有的属性，然后返回所有的验证失败信息)

### 快速失败返回模式（Fast-Fail）
快速失败返回模式(只要有一个验证失败，则返回)。[官网文档](https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/#section-fail-fast) 12.2. Fail fast mode:`Using the fail fast mode, Hibernate Validator allows to return from the current validation as soon as the first constraint violation occurs. This can be useful for the validation of large object graphs where you are only interested in a quick check whether there is any constraint violation at all.`  9.2.8. Provider-specific settings:
```java
ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
        .configure()
        .failFast( true )
        .addMapping( (ConstraintMapping) null )
        .buildValidatorFactory();
Validator validator = validatorFactory.getValidator();
```
Alternatively, provider-specific options can be passed via Configuration#addProperty(). Hibernate Validator supports enabling the fail fast mode that way, too:Enabling a Hibernate Validator specific option via addProperty()
```java
ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
        .configure()
        .addProperty( "hibernate.validator.fail_fast", "true" )
        .buildValidatorFactory();
Validator validator = validatorFactory.getValidator();
```

### SpringBoot中配置
配置hibernate Validator为快速失败返回模式：

```java
@Configuration
public class ValidatorConfiguration {
    @Bean
    public Validator validator(){
        ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
                .configure()
                .addProperty( "hibernate.validator.fail_fast", "true" )
                .buildValidatorFactory();
        Validator validator = validatorFactory.getValidator();

        return validator;
    }
}
```


## group校验
    
### 使用场景         
通常情况下，不同的业务逻辑会有不同的验证逻辑，对UserEntity来说，当更新时id必须不为null，但在增加时id必须为null。所以JSR-303定义了`group概念`，每个校验注解都必须支持。校验注解作用在字段上时，可以指定一个或多个group，当Spring Boot校验对象时，也可以指定校验的上下文属于哪个group，`只有group匹配的时候，校验注解才生效`    

### 分组顺序校验
除了按组指定是否验证之外，还可以指定组的验证顺序，前面组验证不通过的，后面组不进行验证
```java
@GroupSequence({Group1.class, Group1.class, Default.class})
public interface GroupOrder {
}
```

### Example
```java
public class UserEntity {

    @NotNull(groups = { Update.class })
    @Null(groups = { Add.class })
    private Long id;

    @Size(min = 3, max = 20)
    private String userName;

    // 更新时校验组
    public interface Update {
    }

    // 添加时校验组
    public interface Add {
    }
}
```
当校验上下文为Add.class时，@Null生效，id需要为空才能校验通过；当校验上下文为Update.class时@NotNull生效 ，id不能为空

## MVC中使用@Validated or @Valid
只需要在方法参数上@Validated即可触发一次校验，多个参数的，可以加多个@Valid和BindingResult。如下：

#### Bean参数验证
```java
@PostMapping("/addUser")
public void addUser(@Validated({ Add.class }) UserEntity user, BindingResult result) {
    if(result.hasErrors()){
        ...;
    }
}
```

请求参数中使用了@Validated注解，将触发Spring的校验，并将校验的结果放到BindingResult对象中，Validated注解可以使用校验的上下文，整个校验将按照指定的上下文来进行校验  
BindingResult包含了校验结果，提供如下方法：  

- hasErrors 判断校验是否通过
- getAllEErrors 得到所有的错误信息，通常返回的是FieldError列表

如果Controller参数未提供BindingResult对象，则将抛出`MethodArgumentNotValidException`异常

### @RequestParam参数校验

使用bean参数验证的方式，没有办法校验@RequestParam的内容，一般在处理Get请求的时候，会如下处理，但无法校验参数的有效性：

```java
@RequestMapping(value = "/find", method = RequestMethod.GET)
public void find(@RequestParam(name = "ageRange", required = true) String ageRange) {
    
}
```

使用@Valid注解对@RequestParam对应的参数进行校验是无效的，`必须使用@Validated注解，且需要添加到Controller类上，@RequestParam中的required=true，此时使用`MethodValidationPostProcessor` 的Bean（SpringBoot默认提供）：

```java
@Bean
public MethodValidationPostProcessor methodValidationPostProcessor() {
　　 //默认是普通模式，返回所有的验证不通过信息集合
    return new MethodValidationPostProcessor();
}
```

如果需要使用Fast-Fail Model 需要在MethodValidationPostProcessor中设置的Fast-Fail Validator，如下：

```java
@Bean
public MethodValidationPostProcessor methodValidationPostProcessor() {
    MethodValidationPostProcessor postProcessor = new MethodValidationPostProcessor();
　　　　　/**设置validator模式为快速失败返回*/
    postProcessor.setValidator(validator());
    return postProcessor;
}

@Bean
public Validator validator(){
    ValidatorFactory validatorFactory = Validation.byProvider( HibernateValidator.class )
            .configure()
            .addProperty( "hibernate.validator.fail_fast", "true" )
            .buildValidatorFactory();
    Validator validator = validatorFactory.getValidator();

    return validator;
}

```
使用如下：
```java
@RestController
@Validated
public class UserController {

    @GetMapping(value = "findByAge")
    public String findUser(
            @NumberRange(min = 18, max = 32) @RequestParam(name = "ageRange", required = true) String ageRange) {
        return ageRange;
    }

}
```

此时验证不通过时抛出`ConstraintViolationException`异常，可以使用统一异常处理：
```
@ControllerAdvice
@Component
public class ExceptionHandlerConfig {
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseBody
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public String handle(ConstraintViolationException exception) {
        return exception.getLocalizedMessage();
    }
}
```

## 对象级联校验
对象内部包含另一个对象作为属性，属性上加@Valid，可以验证作为属性的对象内部的验证
```java
    @NotNull
    @Valid
    UserEntity
```    

## Custom Validator
JSR-303允许定制校验注解，但需要在自定义注解中添加`@Constraint`来指明具体的校验规则实现类  

自定义校验器注解必须提供如下信息:  
- message 用于创建错误信息，支持表达式，eg "不能大于{max}小时"
- groups 验证规则分组，`验证注解必须提供`
- payload 定义验证规则的有效负荷

### Example

- Validator注解

```java
@Target({ ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NumberRangeValidator.class)
@Documented
/**
 * 
 *
 * @ClassName: NumberRange
 * @Description: 验证数字区间，格式如xx-xx;
 *
 */
public @interface NumberRange {

    String value() default "18-32";

    int min() default 18;

    int max() default 32;

    String message() default "{value}param of illegal";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

}

```

- 校验规则实现

```java
public class NumberRangeValidator implements ConstraintValidator<NumberRange, String> {
    
    //两位整数
    final static String regex = "[0-9]{1,2}-[0-9]{1,2}";

    NumberRange range;

    int min;
    int max;

    @Override
    public void initialize(NumberRange range) {
        // 获取注解的定义
        this.range = range;
        min = range.min();
        max = range.max();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (null == value) {
            return false;
        }
        if (!value.matches(regex)) {
            return false;
        }
        String[] split = value.split("-");
        return Integer.valueOf(split[0]) >= min && Integer.valueOf(split[0]) <= max;
    }

}
```

- 使用注解
```java
@NumberRange(min = 24, max = 45)
private String ageRange;
```



