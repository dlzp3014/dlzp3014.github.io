---
layout: post
title:  "自定义Annotation校验对象中的属性"
date:   2018-12-01 14:02:00
categories: Java
tags: Java Java-Annotation
---

* content
{:toc}




## 校验注解


```java
/**
 * @Description: String类型 参数校验注解
 * @Author: dlzp
 * @Date: 2018/11/25 0025 16:30
 */
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ParamValidAnnotation {

    String regex() default "";

    String errorMsg() default "Param Invalid";

}

```

## 校验器

```java

/**
 * @Description: 验证:对象中一个参数校验失败则返回
 * @Author: dlzp
 * @Date: 2018/11/25 0025 16:38
 */
public class VerifyParamValid {

    private ParamValidAnnotation paramValidAnnotation;

    private Class clazz;

    private String errorMsg;

    private Boolean isSuccess = true;

    public VerifyParamValid(Object object) throws IllegalAccessException {
        clazz = object.getClass();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);//私有变量可访问
            if (!verify(field, object)) {
                isSuccess = false;
                return;
            }
        }
    }

    /**
     * @Description: 验证
     * @param: [field, object]
     * @return: boolean
     */
    private boolean verify(Field field, Object object) throws IllegalAccessException {
        paramValidAnnotation = field.getAnnotation(ParamValidAnnotation.class);
        if (!field.isAnnotationPresent(ParamValidAnnotation.class)) //当前field是否被ParamValidAnnotation注释
            return true;

        Object value = field.get(object);  //获取当前对象属性的值
        String fieldName = field.getName(); //获取当前对象属性的name
        errorMsg = fieldName + ":" + paramValidAnnotation.errorMsg();
        if (null == value)
            return false;

        String regex = paramValidAnnotation.regex();
        if (value.toString().matches(regex))
            return true;
        return false;
    }

    public String getErrorMsg() {
        return errorMsg;
    }

    public void setErrorMsg(String errorMsg) {
        this.errorMsg = errorMsg;
    }

    public Boolean getSuccess() {
        return isSuccess;
    }

    public void setSuccess(Boolean success) {
        isSuccess = success;
    }
}

```

## 校验对象中的属性

```java

/**
 * @Description: 校验
 * @Author: dlzp
 * @Date: 2018/11/25 0025 17:32
 */
public class ValidUtils {

    /**
     * @Description: 验证 ：为null时验证成功
     * @param: [object]
     * @return: java.lang.String
     */
    public static String valid(Object object) throws IllegalAccessException {
        VerifyParamValid paramValid = new VerifyParamValid(object);
        if (paramValid.getSuccess()) {
            return null;
        }
        return paramValid.getErrorMsg();
    }

    public static void main(String[] args) throws IllegalAccessException {
        Person person = new Person();
        person.setName("abcd");
        String valid = valid(person);
        System.out.println(valid == null ? "success" : "errorMsg:"+valid);
    }
}

```











    
