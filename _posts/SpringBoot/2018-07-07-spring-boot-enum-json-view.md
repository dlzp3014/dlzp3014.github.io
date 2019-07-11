---
layout: post
title:  "SpringBoot中Enum枚举返回Json串"
date:   2019-07-11 08:38:00
categories: SpringBoot 
tags: SpringBoot
---

* content
{:toc}


SpringBoot中默认使用Jackson来处理Json串，有时候需要将枚举转换为Json串，具体实现方式如下：



- 使用@JsonFormat注解

在枚举类中标注@JsonFormat，并指定`shape = JsonFormat.Shape.OBJECT`

```java
@JsonFormat(shape = JsonFormat.Shape.OBJECT)
public enum OperationTypeEnum {
    ADD("add"), DELETE("del"), UPDATE("upd"), QUERY("query");
    private String name;

    OperationTypeEnum(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

Result:
{
    "name": "add"
}
```

- 使用@JsonSerialize注解

在枚举类中标注@JsonSerialize注解，并指定using为继承JsonSerializer<OperationTypeEnum>的序列化实现类

```java
public class OperationTypeEnumSerialize extends JsonSerializer<OperationTypeEnum> {

    @Override
    public void serialize(OperationTypeEnum value, JsonGenerator gen, SerializerProvider serializers) throws IOException {
        gen.writeStartObject();
        gen.writeFieldName("name");
        gen.writeString(value.getName());
        gen.writeEndObject();

    }
}

```

- 将自定义的序列化规则注册到ObjectMapper.registerModule(simpleModule)中

```java

ObjectMapper objectMapper = new ObjectMapper();
SimpleModule simpleModule = new SimpleModule();
simpleModule.addSerializer(OperationTypeEnum.class, new OperationTypeEnumSerialize());
objectMapper.registerModule(simpleModule);
System.out.println(objectMapper.writeValueAsString(OperationTypeEnum.ADD)); //{"name":"add"}


```
