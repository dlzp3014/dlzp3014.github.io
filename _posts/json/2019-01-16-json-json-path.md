---
layout: post
title: "json-path使用说明"
date: 2019-01-15 08:38:00
categories: JSON 
tags: JsonPath
---

* content
{:toc}


`Jayway JsonPath`是一个用于读取JSON文档的`Java DSL`类库，表达式总是引用JSON结构，其方式与XPath表达式与XML文档结合使用的方式相同。不管对象还是数组，JsonPath中的根元素总是被引用为`$`，可以使用点符号`$.store.book[0].title`或者括号符号`$['store']['book'][0]['title']`。[GitHub地址为:https://github.com/json-path/JsonPath](https://github.com/json-path/JsonPath)





## 操作符


|操作符 |描述 |
| ---- | ---- |
|$|要查询的根元素，所有路径表达式的开始|
|@|通过过滤谓词处理当前节点|
|*|通配符，任何需要名称或数字的地方都可以使用|
|..|深度扫描，任何需要名称的地方都可以使用|
|.\<name>|点标注的子节点|
|['\<name>' (, '\<name>')]|有括号的子结点或子结点|
|[\<number> (, \<number>)]|数组索引|
|[start:end]|数组切片操作符|
|[?(\<expression>)]|过滤表达式，表达式的值必须为布尔值|

## 函数

函数可以在路径的末尾调用 —— 函数的输入是路径表达式的输出，函数输出由函数本身决定

|函数|描述|输出|
| ---- | ---- | ----|
|min()|提供数字数组的最小值|double类型|
|max()|提供数字数组的最大值|double类型|
|avg()|提供数字数组的评价值|double类型|
|stddev()|提供数字数组的标准差|double类型|
|length()|提供数组的长度|Integer类型|
|sum()|提供数字数组的和|double类型|


## 过滤操作符

过滤器是用于过滤数组的逻辑表达式。通常过滤应该是`[?(@.age > 18)]`，`@`代表当前处理的元素。更多复杂的过滤器可以使用逻辑操作符`&&、||`创建，字符串字面量必须使用单引号或者双引号括起来` ([?(@.color == 'blue')] or [?(@.color == "blue")]).`


|操作符 |描述 |
| ---- | ---- |
|==|左边等于右边(注意1不等于'1')|
|!=|左边不等于右边|
|<|左边小于右边|
|<=|左边小于或等于右边|
|>|左边大于右边|
|>=|左边大于或等于右边|
|=~|左匹配正则表达式`[?(@.name =~ /foo.*?/i)]`|
|in|左边存在于右边`[?(@.size in ['S', 'M'])]`|
|nin|左边不存在于右边|
|subsetof|左是右的子集`[?(@.sizes subsetof ['S', 'M', 'L'])]`|
|anyof|左边和右边相交`[?(@.sizes anyof ['M', 'L'])]`|
|noneof|左边和右边没有交集`[?(@.sizes noneof ['M', 'L'])]`|
|size|左边数组或字符串大小匹配右边|
|empty|左边数组或字符为空|


## 路径示例

```json
{
    "store": {
        "book": [
            {
                "category": "reference",
                "author": "Nigel Rees",
                "title": "Sayings of the Century",
                "price": 8.95
            },
            {
                "category": "fiction",
                "author": "Evelyn Waugh",
                "title": "Sword of Honour",
                "price": 12.99
            },
            {
                "category": "fiction",
                "author": "Herman Melville",
                "title": "Moby Dick",
                "isbn": "0-553-21311-3",
                "price": 8.99
            },
            {
                "category": "fiction",
                "author": "J. R. R. Tolkien",
                "title": "The Lord of the Rings",
                "isbn": "0-395-19395-8",
                "price": 22.99
            }
        ],
        "bicycle": {
            "color": "red",
            "price": 19.95
        }
    },
    "expensive": 10
}
```

JsonPath (click link to try)	Result
- $.store.book[*].author	The authors of all books
- $..author	All authors
- $.store.*	All things, both books and bicycles
- $.store..price	The price of everything
- $..book[2]	The third book
- $..book[-2]	The second to last book
- $..book[0,1]	The first two books
- $..book[:2]	All books from index 0 (inclusive) until index 2 (exclusive)
- $..book[1:2]	All books from index 1 (inclusive) until index 2 (exclusive)
- $..book[-2:]	Last two books
- $..book[2:]	Book number two from tail
- $..book[?(@.isbn)]	All books with an ISBN number
- $.store.book[?(@.price < 10)]	All books in store cheaper than 10
- $..book[?(@.price <= $['expensive'])]	All books in store that are not "expensive"
- $..book[?(@.author =~ /.*REES/i)]	All books matching regex (ignore case)
- $..*	Give me every thing
- $..book.length()	The number of books


- maven依赖:

```xml
<dependency>
    <groupId>com.jayway.jsonpath</groupId>
    <artifactId>json-path</artifactId>
    <version>2.4.0</version>
</dependency>
```

- 读取文档:

JsonPath#read(String json, String jsonPath, Predicate... filters):创建一个新的JsonPath并将其应用于提供的Json字符串

```json
String jsonStr = "{'name':'dlzp'}";
String name = JsonPath.read(jsonStr, "$.name");
Assert.assertEquals("dlzp", name);
```

`每次调用JsonPath.read(…)时都会解析文档，为了避免这个问题可以先解析json`

```java

String jsonStr = "{'name':'dlzp'}";
Object document = Configuration.defaultConfiguration().jsonProvider().parse(jsonStr);//解析

//多次读取
Assert.assertEquals("dlzp", JsonPath.read(document, "$.name"));
Assert.assertEquals("dlzp", JsonPath.read(document, "$.name"));
```

也可以使用JsonPath提供的`fluent API`:

```java
String jsonStr = "{'name':'dlzp'}";

String name = JsonPath
        .using(Configuration.defaultConfiguration())
        .parse(jsonStr)
        .read("$.name", String.class);
Assert.assertEquals("dlzp", name);
```

在java中使用JsonPath时，重要的是要知道希望在结果中使用哪种类型，JsonPath将自动尝试将结果转换为期望的类型。在评估一个路径时，需要理解什么时候路径是确定的。当一个路径是不确定的时它包含:

```json
.. - a deep scan operator 深度扫描
?(<expression>) - an expression
[<number>, <number> (, <number>)] - multiple array indexes 多个数组索引
```
不定路径总是返回一个列表(由当前JsonProvider表示)

默认情况下，一个简单的对象映射器是由`MappingProvider SPI`提供，可以指定的MappingProvider服务提供者(默认使用JsonSmartJsonProvider)。如果JsonPath配置了`JacksonMappingProvider` or `GsonMappingProvider`，可以将JsonPath输出直接映射到POJO中：

```java
String jsonStr = "[ {\n" +
                "        \"name\": \"dlzp\"\n" +
                "    }\n" +
                "]";

Person person = JsonPath.using(Configuration
        .builder()
        .jsonProvider(new JacksonJsonProvider())
        .mappingProvider(new JacksonMappingProvider())
        .build()
).parse(jsonStr).read("$[0]", Person.class);

Assert.assertEquals("dlzp", person.getName());
```

使用`TypeRef`可以获取完整的泛型类型信息：

```java
String jsonStr = "[ {\n" +
                "        \"name\": \"dlzp\"\n" +
                "    }\n" +
                "]";

TypeRef<Person> typeRef = new TypeRef<Person>() {
};
Person person = JsonPath.using(Configuration
        .builder()
        .jsonProvider(new JacksonJsonProvider())
        .mappingProvider(new JacksonMappingProvider())
        .build()
).parse(jsonStr).read("$[0]", typeRef);

Assert.assertEquals("dlzp", person.getName());
```

- Predicates谓词

在JsonPath中创建过滤谓词有三种不同的方法

1).内联谓词

```java
String jsonStr = "{\n" +
                "    \"user\": {\n" +
                "        \"name\": \"dlzp\", \n" +
                "        \"age\": 32\n" +
                "    }\n" +
                "}";

List<Map<String, Object>> read = JsonPath.parse(jsonStr)
        .read("$.user[?(@.age > 20)]");

Assert.assertEquals("dlzp", read.get(0).get("name"));
```

也可以使用`&&、||`组合多个谓词或者`!`取反操作

```java
 [?(@.price < 10 && @.category == 'fiction')] , [?(@.category == 'reference' || @.price > 10)].

[?(!(@.price < 10 && @.category == 'fiction'))].
```

2).过滤谓词

`Filter API`构建谓词

```java
String jsonStr = "{\n" +
                "    \"user\": {\n" +
                "        \"name\": \"dlzp\", \n" +
                "        \"age\": 32\n" +
                "    }\n" +
                "}";

List<Map<String, Object>> read = JsonPath.parse(jsonStr)
        .read("$.user[?]", Filter
                .filter(Criteria.where("name").is("dlzp").and("age").gte(30)));

Assert.assertEquals("dlzp", read.get(0).get("name"));
```

占位符`?`为路径中的过滤器。`提供多个过滤器时，它们的应用顺序是占位符的数量必须与提供的筛选器的数量匹配`，可以在一个过滤器操作中指定多个谓词占位符。过滤器也能组合` 'OR' and 'AND'`

```java
Filter fooOrBar = filter(
   where("foo").exists(true)).or(where("bar").exists(true)
);
   
Filter fooAndBar = filter(
   where("foo").exists(true)).and(where("bar").exists(true)
);
```

3).实现Predicate接口

```java
Predicate booksWithISBN = new Predicate() {
    @Override
    public boolean apply(PredicateContext ctx) {
        return ctx.item(Map.class).containsKey("isbn");
    }
};

List<Map<String, Object>> books = 
   reader.read("$.store.book[?].isbn", List.class, booksWithISBN);

List<Map<String, Object>> read = JsonPath.parse(jsonStr)
                .read("$.user[?]", ctx -> ctx.item(Map.class).containsKey("name"));
```

- 获取属性的路径

```java
Configuration conf = Configuration.builder()
   .options(Option.AS_PATH_LIST).build();

List<String> pathList = using(conf).parse(json).read("$..author");

assertThat(pathList).containsExactly(
    "$['store']['book'][0]['author']",
    "$['store']['book'][1]['author']",
    "$['store']['book'][2]['author']",
    "$['store']['book'][3]['author']");

List<String> read = JsonPath.using(conf).parse(jsonStr).read("$..name");
System.out.println(read); //["$['user']['name']"]

```

## Configuration对象配置

- 参数选项

1).DEFAULT_PATH_LEAF_TO_NULL:使JsonPath对于丢失的叶子返回null

```java
[
   {
      "name" : "john",
      "gender" : "male"
   },
   {
      "name" : "ben"
   }
]

Configuration conf = Configuration.defaultConfiguration();

//Works fine
String gender0 = JsonPath.using(conf).parse(json).read("$[0]['gender']");
//PathNotFoundException thrown
String gender1 = JsonPath.using(conf).parse(json).read("$[1]['gender']");

Configuration conf2 = conf.addOptions(Option.DEFAULT_PATH_LEAF_TO_NULL);

//Works fine
String gender0 = JsonPath.using(conf2).parse(json).read("$[0]['gender']");
//Works fine (null is returned)
String gender1 = JsonPath.using(conf2).parse(json).read("$[1]['gender']");
```

2).ALWAYS_RETURN_LIST:将JsonPath配置为返回一个列表，`即使该路径是definite确定的`

```java
Configuration conf = Configuration.defaultConfiguration();

//Works fine
List<String> genders0 = JsonPath.using(conf).parse(json).read("$[0]['gender']");
//PathNotFoundException thrown
List<String> genders1 = JsonPath.using(conf).parse(json).read("$[1]['gender']");
```

3).SUPPRESS_EXCEPTIONS:确保不会从路径计算传播任何异常

如果存在ALWAYS_RETURN_LIST选项，则返回一个空列表;如果选项ALWAYS_RETURN_LIST不存在null返回

- JsonProvider SPI

JsonPath有五种不同的jsonprovider:JsonSmartJsonProvider (default)、JacksonJsonProvider、JacksonJsonNodeJsonProvider、GsonJsonProvider、JsonOrgJsonProvider

只有在初始化应用程序时，才能更改配置默认值，`强烈建议不要在运行时进行更改，特别是在多线程应用程序中`

```java
Configuration.setDefaults(new Configuration.Defaults() {

    private final JsonProvider jsonProvider = new JacksonJsonProvider();
    private final MappingProvider mappingProvider = new JacksonMappingProvider();
      
    @Override
    public JsonProvider jsonProvider() {
        return jsonProvider;
    }

    @Override
    public MappingProvider mappingProvider() {
        return mappingProvider;
    }
    
    @Override
    public Set<Option> options() {
        return EnumSet.noneOf(Option.class);
    }
});
```

Note that the JacksonJsonProvider requires `com.fasterxml.jackson.core:jackson-databind:2.4.5 and the GsonJsonProvider requires com.google.code.gson:gson:2.3.1 on your classpath`

- Cache SPI

允许API使用者以适合自己需要的方式配置路径缓存，在第一次访问缓存或抛出JsonPathException异常之前，必须对缓存进行配置。JsonPath附带两个缓存实现：
`com.jayway.jsonpath.spi.cache.LRUCache (default, thread safe)`

`com.jayway.jsonpath.spi.cache.NOOPCache (no cache)`

简单实现自定义缓存：

```java
CacheProvider.setCache(new Cache() {
    //Not thread safe simple cache
    private Map<String, JsonPath> map = new HashMap<String, JsonPath>();

    @Override
    public JsonPath get(String key) {
        return map.get(key);
    }

    @Override
    public void put(String key, JsonPath jsonPath) {
        map.put(key, jsonPath);
    }
});
```