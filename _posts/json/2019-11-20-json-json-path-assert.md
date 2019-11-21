---
layout: post
title: "json-path-assert使用说明"
date: 2019-11-20 08:38:00
categories: JSON 
tags: JsonPath
---

* content
{:toc}


json-path-assert是一个带有`hamcrest-matchers`的JsonPath类库。GitHub地址:https://github.com/json-path/JsonPath/tree/master/json-path-assert

Maven依赖如下:

```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path-assert</artifactId>
    <version>2.2.0</version>
</dependency>
```

使用示例(来自JsonPath-assert:README.md):




```java
//静态导包
import static com.jayway.jsonpath.matchers.JsonPathMatchers.*;
import static org.junit.Assert.*;

// 是否为json
assertThat(json, isJson());

// 是否包含Json路径
assertThat(json, hasJsonPath("$.message"));
assertThat(json, hasNoJsonPath("$.message"));

// 校验JSON值
assertThat(json, hasJsonPath("$.message", equalTo("Hi there")));
assertThat(json, hasJsonPath("$.quantity", equalTo(5)));
assertThat(json, hasJsonPath("$.price", equalTo(34.56)));
assertThat(json, hasJsonPath("$.store.book[*].author", hasSize(4)));
assertThat(json, hasJsonPath("$.store.book[*].author", hasItem("Evelyn Waugh")));
```

- 组合多个Matchers 


```java
assertThat(json, isJson(withoutJsonPath("...")));
assertThat(json, isJson(withJsonPath("...", equalTo(3)))); 

//将多个JSON路径计算合并到一个语句中
assertThat(json, isJson(allOf(
    withJsonPath("$.store.name", equalTo("Little Shop")),
    withoutJsonPath("$.expensive"),
    withJsonPath("$..title", hasSize(4)))));

```


- 匹配预编译的复杂JSON路径表达式

```java
//过滤
Filter cheapFictionFilter = filter(
    where("category").is("fiction").and("price").lte(10D));
JsonPath cheapFiction = JsonPath.compile("$.store.book[?]", cheapFictionFilter);
String json = ...;
assertThat(json, isJson(withJsonPath(cheapFiction)));
```

- 类型匹配

```java
String json = ...
//jsonString
assertThat(json, isJsonString(withJsonPath("$..author")));

File json = ...
assertThat(json, isJsonFile(withJsonPath("$..author")));
```


- 不确定路径

当使用不确定路径表达式` wildcards '*'`，其结果将产生一个列表，如果没有匹配到的实体，则可能为空列表。如果需要` assert that`列表的值中包含某些元素时，一定要明确的表达出来:

```java
{
  "items": []
}

// Both of these statements will succeed(!)
assertThat(json, hasJsonPath("$.items[*]"));
assertThat(json, hasJsonPath("$.items[*].name"));

// Make sure to explicitly check for size if you want to catch this scenario as a failure
assertThat(json, hasJsonPath("$.items[*]", hasSize(greaterThan(0))));

// However, checking for the existence of an array works fine, as is
assertThat(json, hasJsonPath("$.not_here[*]"));
```

- Json中的null

`null`是一个有效的JSON值。如果存在这样的值，则该路径仍然被认为是有效路径

```java
// Given a JSON like this:
{ "none": null }

// All of these will succeed, since '$.none' is a valid path
assertThat(json, hasJsonPath("$.none"));
assertThat(json, isJson(withJsonPath("$.none")));
assertThat(json, hasJsonPath("$.none", nullValue()));
assertThat(json, isJson(withJsonPath("$.none", nullValue())));

// But all of these will fail, since '$.not_there' is not a valid path
assertThat(json, hasJsonPath("$.not_there"));
assertThat(json, isJson(withJsonPath("$.not_there")));
assertThat(json, hasJsonPath("$.not_there", anything()));
assertThat(json, isJson(withJsonPath("$.not_there", anything())));
```