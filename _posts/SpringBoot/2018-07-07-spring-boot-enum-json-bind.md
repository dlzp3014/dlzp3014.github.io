---
layout: post
title:  "SpringBoot中通用Enum枚举值绑定"
date:   2019-07-11 08:38:00
categories: SpringBoot 
tags: SpringBoot
---

* content
{:toc}


- 在面对`Content-Type=application/x-www-form-urlencoded`时的@GetMapping等请求时，SpringBoot默认使用StringToEnumConverterFactory提供的StringToEnum<T extends Enum>将Enum<T># String name()值作为绑定依据，来实现对枚举的参数绑定，如下：


```java
@GetMapping("/bind/{operation}") //operation占位符参数必须为OperationTypeEnum中的name值
public OperationTypeEnum operationTypeEnum(@PathVariable OperationTypeEnum operation) {
    return operation;
}

public enum OperationTypeEnum {
    ADD, DELETE, UPDATE, QUERY;
}

```
具体请参考:[Spring源码-ConverterFactory<S, R>转换器工厂](/2019/07/11/spring-boot-enum-json-view/)，依据其提供的实现，可以自定义通用枚举的转换方式


- 当`Content-Type=application/json`时SpringBoot默认使用jackson提供ObjectMapper类进行转换，即依据Enum<T\># String name()值，如下

```java
@PostMapping("/bind/operation")
public SysUser operationTypeEnumPost(@RequestBody SysUser sysUser) {
    return sysUser;
}
public class SysUser {
    private OperationTypeEnum operationType;
}

RequestBody:
{
	
	"operationType":"ADD"
}
```

此时就需要自定义jackson的反序列化策略






## application/x-www-form-urlencoded处理

### 定义通用枚举接口

```java
/**
* T：枚举类型
* I: 枚举转换标识符类型
*/
public interface BaseEnum<T extends Enum<T>> {

    /**
     * 枚举转换唯一标识
     */
    I getId();

    /**
     * 属性名
     */
    @JsonIgnore
    default String getAttr() {
        return "id";
    }

}
```

### 通用枚举转换工厂实现

```java
/**
 * 基于BaseEnum接口实现的枚举转换接口
 */
public class BaseEnumConvertFactory implements ConverterFactory<String, BaseEnum> {

    private final Map<Class, Converter> converterCache = new WeakHashMap<>();

    @Override
    public <T extends BaseEnum> Converter<String, T> getConverter(Class<T> targetType) {

        return converterCache.computeIfAbsent(targetType, k -> converterCache.put(k, source -> Stream.of(targetType.getEnumConstants())
                .filter(ele -> source.equals(ele.getId().toString()))
                .findAny()
                .orElseGet(null)));
    }
}



```

### 注册通用枚举转换工厂

```java
@Configuration
public class MvcConfiguration implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverterFactory(new BaseEnumConvertFactory());
    }

}
```

### 自定义枚举实现

```java
public enum GenderEnum implements BaseEnum<GenderEnum, Integer> {

    MALE(1, "male"), FEMALE(2, "female");

    private Integer id;

    private String value;

    GenderEnum(Integer id, String value) {
        this.id = id;
        this.value = value;
    }

    public String getValue() {
        return value;
    }

    @Override
    public Integer getId() {
        return id;
    }

}


```

### @GetRequest接口请求

```java
@GetMapping("/bind/{genderId}")
public GenderEnum binderPath(@PathVariable(name = "genderId") GenderEnum gender) {

    return gender;
}

@GetMapping("/bind/genderEnum")
public GenderEnum binderParam(@RequestParam(name = "genderId") GenderEnum gender) {

    return gender;
}


```




## application/json处理

### Jackson反序列化实现

```java
public class BaseEnumDeserialize<T extends BaseEnum> extends JsonDeserializer<T> implements ContextualDeserializer {

	//枚举类型
    Class<T> target;

    @Override
    public T deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException, JsonProcessingException {

        JsonNode node = jsonParser.getCodec().readTree(jsonParser);
        for (T enumConstant : target.getEnumConstants()) {
        	//枚举中唯一标识的属性名
            if (node.get(enumConstant.getAttr()).asText().equals(enumConstant.getId().toString())) {
                return enumConstant;
            }
        }
        return null;
    }

    @Override
    public JsonDeserializer<T> createContextual(DeserializationContext ctxt, BeanProperty property) throws JsonMappingException {
        // ctxt.setAttribute(property.getName(), property.getType().getRawClass());
        target = (Class<T>) ctxt.getContextualType().getRawClass();
        return this;
    }
```

### 使用@JsonDeserialize标注

```java
@JsonDeserialize(using = BaseEnumDeserialize.class) 
public interface BaseEnum<T extends Enum<T>, I> {

}
```

### @PostRequest接口请求

```java

RequestBody:
{
	"id":3
}
@PostMapping("/bind/GenderEnum")
public GenderEnum binderGenderEnum(@RequestBody GenderEnum gender) {

    return gender;
}

RequestBody:

{
	"genderEnum":{"id":1}
}
@PostMapping("/binder/GenderEnum")
public SysUser binderGenderEnum(@RequestBody SysUser sysUser) {

    return sysUser;
}

public class SysUser {
    ......
    private GenderEnum genderEnum;
    ......
}

```