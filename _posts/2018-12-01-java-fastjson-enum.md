---
layout: post
title:  "FastJson自定义枚举类型的序列化与反序列化"
date:   2018-12-01 14:02:00
categories: FastJson
tags: FastJson
---

* content
{:toc}
FastJson在序列化与反序列化枚举类型时，通过会有如下做法：
- 根据枚举的name值【默认】
- 根据枚举的ordinal值
- 根据枚举的toString方法，反序列化时仍采用枚举的name值
以上做法都不够灵活，因此可以使用@JSONField中的serializeUsing与deserializeUsing属性来指定序列化和反序列化的实现来完成。[官网说明](https://github.com/alibaba/fastjson/wiki/JSONField)



## FastJson对enum序列化、反序列化支持
fastjon默认对enum对象使用WriteEnumUsingName属性，因此会将enum值序列化为其Name，使用WriteEnumUsingToString方法可以序列化时将Enum转换为toString()的返回值；
同时override toString函数能够将enum值输出需要的形式。但是这样做会带来一个问题，对应的反序列化使用的Enum的静态方法valueof可能无法识别自行生成的toString()，导致反序列化出错；
如果将节省enum序列化后的大小，可以将enum序列化其ordinal值，保存为int类型。fastJson在反序列化时，如果值为int，则能够使用ordinal值匹配，找到合适的对象。fastjson要将enum序列化为ordinal只需要禁止WriteEnumUsingName feature，首先根据默认的features排除WriteEnumUsingName,然后使用新的features序列化即可。

```java
Assert.assertEquals("\"FEMALE\"",JSON.toJSONString(GenderEnum.FEMALE));
Assert.assertEquals("1",JSON.toJSONString(GenderEnum.FEMALE,
        SerializerFeature.config(
                JSON.DEFAULT_GENERATE_FEATURE,
                SerializerFeature.WriteEnumUsingName, false)));
JSON.toJSONString(GenderEnum.FEMALE, SerializerFeature.WriteEnumUsingToString);//"{id:2, gender:'female'}"

//enum as JavaBean
SerializeConfig config=new SerializeConfig();
config.configEnumAsJavaBean(GenderEnum.class);
JSON.toJSONString(GenderEnum.FEMALE, config);//{"gender":"female","id":2}
//enum 全部值
JSON.toJSONString(GenderEnum.values(), config);//[{"gender":"male","id":1},{"gender":"female","id":2}]
```



## 自定义枚举类型编解码器

```java
/**
 * @Description: 枚举
 * @Author: dlzp
 * @Date: 2018/12/2 0002 18:30
 */
public enum GenderEnum {
    MALE(1, "male"), FEMALE(2, "female");

    private Integer id;
    private String gender;

    GenderEnum(Integer id, String gender) {
        this.id = id;
        this.gender = gender;
    }

    public static GenderEnum getInstance(Integer id) {

        for (GenderEnum gender : GenderEnum.values()) {
            if (gender.getId().equals(id)) {
                return gender;
            }
        }
        return null;
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

}


/**
 * @Description: 自定义枚举类型的编、解码器
 * @Author: dlzp
 * @Date: 2018/12/2 0002 18:40
 */
public class GenderCodec implements ObjectSerializer, ObjectDeserializer {

    @Override
    public <T> T deserialze(DefaultJSONParser parser, Type type, Object o) {
        //Object value = parser.parse();
        Integer value = parser.parseObject(Integer.class);
        return null == value? null : (T) GenderEnum.getInstance(TypeUtils.castToInt(value));
    }

    @Override
    public int getFastMatchToken() {
        return JSONToken.LITERAL_INT;
    }

    @Override
    public void write(JSONSerializer jsonSerializer, Object o, Object o1, Type type, int i) throws IOException {
        if (null==o){
            jsonSerializer.writeNull();
            return;
        }
        jsonSerializer.write(((GenderEnum) o).getId());
    }
}

public class Person {

    @JSONField(deserializeUsing = GenderCodec.class, serializeUsing = GenderCodec.class)
    private GenderEnum gender;

    public GenderEnum getGender() {
        return gender;
    }

    public void setGender(GenderEnum gender) {
        this.gender = gender;
    }

    @Test
    public void test(){
        Person person=new Person();
        person.setGender(GenderEnum.FEMALE);
        String jsonString = JSON.toJSONString(person);
        System.out.println("jsonStr"+jsonString);
        Person result = JSON.parseObject(jsonString, Person.class);
        System.out.println("Person:"+result.getGender().getGender());
    }
}

```

## 注册全局编解码器

避免每次在属性中添加@JSONField注解，SpringBoot中可以使用@Component和PostConstruct注解，在初始化Bean之前执行

```java
@Component
public class FastJsonConfig{
    @PostConstruct
    public void init(){
        SerializeConfig.getGlobalInstance().put(GenderEnum.class, new GenderCodec());
        ParserConfig.getGlobalInstance().putDeserializer(GenderEnum.class, new GenderCodec());
    }
}

```



