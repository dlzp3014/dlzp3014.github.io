---
layout: post
title:  "Spring RestTemplate详细介绍"
date:   2018-08-07 22:56:00
categories: Spring 
tags: Spring
---

* content
{:toc}

RestTemplate主要用于便编写与REST资源进行交互，涵盖了除了Trace外所有的HTTP动作，RestTemplate定义了11个独立的操作，每一个操作都有重载.   
[Java API](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html),[官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#rest-client-access)。方法说明如下：





|方法|描述|
|:------    |:------|
|delete     |在特定的URL上对资源执行HTTP DELETE 操作    |
|exchange   |在URL上执行特定的HTTP 方法，返回包含对象的ResponseEntity，这个对象是从响应体中映射得到的          |
|execute    |在URL上执行特定的HTTP 方法，返回一个从响应体映射得到的对象         |
|getForObject|发送一个HTTP GET请求，返回的响应体将映射为一个对象           |
|getForEntity    | 发送一个HTTP GET请求，返回的ResponseEntity包含了响应体所映射成的对象          |
|headForHeaders    |发送HTTP HEADER请求，返回包含特定资源URL的HTTP头          |
|optionsForAllow    |发送HTTP OPTIONS请求，返回特定URL的Allow头信息           |
|postForEntity    |POST数据到一个URL,返回包含一个对象的ResponseEntity，这个对象是从响应体中映射得到的          |
|postForLocation    |POST数据到一个URL，返回新创建资源的URL           |
|postForObject    |POST数据到一个URL,返回根据响应体匹配形成的对象           |
|put    |PUT资源到特定的URL|


## GET 资源
执行GET请求有两种方式：getForObject、getForEntity，其中每个方法都有三种形式的重载   
getForObject方法签名如下：

    getForObject(String, Class<T>, Object...)
    getForObject(String, Class<T>, Map<String, ?>)
    getForObject(URI, Class<T>)
    
getForEntity方法签名如下：

    getForEntity(String, Class<T>, Object...)
    getForEntity(String, Class<T>, Map<String, ?>)
    getForEntity(URI, Class<T>)

getForObject、getForEntity都执行根据URL检索资源的GET请求，它们都将资源根据responseType参数匹配为一定的类型，唯一的区别在于getForObject()只返回所请求类型的对象，而getForEntity()方法返回请求的对象以及相应的额外信息

### 检索资源

请求一个资源并按照所选择的Java类型接收该资源，如下：

```java

public User getRemoteUser(String id) {
    RestTemplate rest = new RestTemplate();
    return rest.getForObject(".../api/user/{userId}", User.class, id);
}

```
RestTemplate可以接收参数化URL，如URL中的{userId}占位符最终将会用方法的id参数来填充。getForObject方法的最后一个参数是大小可变的参数列表，每个参数都会按出现顺序插入到指定的URL的占位符中；另一种代替方案是将id参数放到Map中，并以id作为key，然后将这个Map作为最有一个参数传递给getForObject()

```java

public User getRemoteUser(String id) {
    Map<String, String> urlVariables = Collections.singletonMap("id", id);
    RestTemplate rest = new RestTemplate();
    return rest.getForObject(".../api/user/{userId}", User.class, urlVariables);
}

```
getForObject()方法抛出的异常都是非检查型的，如果在getForObject()中有错误，将抛出非检查型RestClientException异常，可以捕获该异常做进一步处理

### 获取响应的元数据
getForObject()只会返回资源(通过HTTP信息转换器将其转换为Java对象)，而getForEntity()会在ResponseEntity中返回相同的对象，且ResponseEntity还带有关于响应的额外信息，如HTTP状态码和响应头

```java
//服务端在header中提供了LastModified信息，资源的最后修改时间
Date lastModified=new Date(response.getHeaders().getLastModified());


//获取状态码
public HttpStatus getStatusCode() {
    if (this.status instanceof HttpStatus) {
        return (HttpStatus) this.status;
    }
    else {
        return HttpStatus.valueOf((Integer) this.status);
    }
}

```

## PUT资源

RestTemplate提供可3种形式的PUT方法：

```java

@Override
public void put(String url, @Nullable Object request, Object... uriVariables)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request);
    execute(url, HttpMethod.PUT, requestCallback, null, uriVariables);
}

@Override
public void put(String url, @Nullable Object request, Map<String, ?> uriVariables)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request);
    execute(url, HttpMethod.PUT, requestCallback, null, uriVariables);
}

@Override
public void put(URI url, @Nullable Object request) throws RestClientException {
    RequestCallback requestCallback = httpEntityCallback(request);
    execute(url, HttpMethod.PUT, requestCallback, null);
}
```

- 使用URI作为参数时，要进行字符串拼接
- 基于String作为参数时，可以将URI指定位模板并对可变部分插入值

```java
public User putRemoteUser(User user) {
    RestTemplate rest = new RestTemplate();
    return rest.put(".../api/user/{userId}", user, user.getId());
}
```
- 当RestTemplate发生PUT请求时，URI模板将{userId}部分用user.getId()方法的返回值来进行替换
- 当使用Map来传递模板参数时，Map的每个key值域RUI模板中占位符变量的名称相同
- put()方法中的第二个参数都是表示自愿的Java对象，它将按照指定的URI发生到服务器端。
- 对象将被转换成什么样的类型很大程度上取决于传递给put()方法的类型：
    给定一个String值时，将会使用*StringHttpMessageConverter*:这个值直接被写到请求体中，内容类型设置为“text/plain”
    给定一个MultiValueMap<String,String>,Map中的值将会被*FormHttpMessageConverter*以“application/x-www-form-urlencoded”的格式写到请求体中
    能够处理任意对象的信息转换器，如果类路径下包含jackson2库，将使用*MappingJackson2HttpMessageConverter*以“application/json”格式将Object写到请求中
    
## DELETE资源

删除服务端的某个或者某些资源，可以调用RestTemplate的delete()方法，签名如下：

```java
@Override
public void delete(String url, Object... uriVariables) throws RestClientException {
    execute(url, HttpMethod.DELETE, null, null, uriVariables);
}

@Override
public void delete(String url, Map<String, ?> uriVariables) throws RestClientException {
    execute(url, HttpMethod.DELETE, null, null, uriVariables);
}

@Override
public void delete(URI url) throws RestClientException {
    execute(url, HttpMethod.DELETE, null, null);
}
```

使用如下：

```java
public void deleteRemoteUser(String userId) {
    RestTemplate rest = new RestTemplate();
    return rest.delete(".../api/user/{userId}", userId);
}
```

## POST资源

在使用RestTemplate来POST一个新的对象时，由于服务端并不存在它，所以它不是一个真正的REST资源，也没有RUL。另外，在服务端创建之前，客户端并不知道对象的ID

### 在POST请求中获取响应对象

POST资源到服务端的一种方式是使用RestTemplate的postForObject()方法，postForObject()方法有三个变种签名：

```java

@Override
@Nullable
public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType,
        Object... uriVariables) throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    HttpMessageConverterExtractor<T> responseExtractor =
            new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
    return execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables);
}

@Override
@Nullable
public <T> T postForObject(String url, @Nullable Object request, Class<T> responseType,
        Map<String, ?> uriVariables) throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    HttpMessageConverterExtractor<T> responseExtractor =
            new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
    return execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables);
}

@Override
@Nullable
public <T> T postForObject(URI url, @Nullable Object request, Class<T> responseType)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    HttpMessageConverterExtractor<T> responseExtractor =
            new HttpMessageConverterExtractor<>(responseType, getMessageConverters());
    return execute(url, HttpMethod.POST, requestCallback, responseExtractor);
}
```

所有方法的第一个参数都是资源要POST到的URL，第二个参数时要发送的对象，而第三个参数时预期返回的Java类型，在将URL作为String类型的两个版本中，第四个参数指定了URL变量（可变参数或者Map）

使用如下：

```java
public User postRemoteUser(User user) {
    RestTemplate rest = new RestTemplate();
    return rest.postForObject(".../api/user", user,User.class);
}
```

如果想得到请求带回来的一些元数据，可以用postForEntity()方法：

```java
@Override
public <T> ResponseEntity<T> postForEntity(String url, @Nullable Object request,
        Class<T> responseType, Object... uriVariables) throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);
    return nonNull(execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables));
}

@Override
public <T> ResponseEntity<T> postForEntity(String url, @Nullable Object request,
        Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);
    return nonNull(execute(url, HttpMethod.POST, requestCallback, responseExtractor, uriVariables));
}

@Override
public <T> ResponseEntity<T> postForEntity(URI url, @Nullable Object request, Class<T> responseType)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request, responseType);
    ResponseExtractor<ResponseEntity<T>> responseExtractor = responseEntityExtractor(responseType);
    return nonNull(execute(url, HttpMethod.POST, requestCallback, responseExtractor));
}
```
如下获取响应中Location头信息的值：

```java
public User postRemoteUser(User user) {
    RestTemplate rest = new RestTemplate();
    
    ResponseEntity<User> response=rest.postForEntity(".../api/user", user,User.class);
    URI url=response.getHeaders().getLocation();
    
    return response.getBody();
}
```

### 在POST请求后获取资源位置

RestTemplate中的postForLocation()方法会在POST请求的请求体中发送一个资源到服务器端，但是响应不再是相同的资源对象，postForLocation()的响应时新创建资源的位置，有如下三个方法签名：

```java
@Override
@Nullable
public URI postForLocation(String url, @Nullable Object request, Object... uriVariables)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request);
    HttpHeaders headers = execute(url, HttpMethod.POST, requestCallback, headersExtractor(), uriVariables);
    return (headers != null ? headers.getLocation() : null);
}

@Override
@Nullable
public URI postForLocation(String url, @Nullable Object request, Map<String, ?> uriVariables)
        throws RestClientException {

    RequestCallback requestCallback = httpEntityCallback(request);
    HttpHeaders headers = execute(url, HttpMethod.POST, requestCallback, headersExtractor(), uriVariables);
    return (headers != null ? headers.getLocation() : null);
}

@Override
@Nullable
public URI postForLocation(URI url, @Nullable Object request) throws RestClientException {
    RequestCallback requestCallback = httpEntityCallback(request);
    HttpHeaders headers = execute(url, HttpMethod.POST, requestCallback, headersExtractor());
    return (headers != null ? headers.getLocation() : null);
}
```

如下在返回中包含资源的URL:

```java
public String postRemoteUser(User user) {
    
    RestTemplate rest = new RestTemplate();
    
    return rest.postForLocation(".../api/user", user).toString();
}
```
在创建资源后，如果服务端在响应的Location头信息中返回新资源的URL,接下来postForLocation()会以String的格式返回该URL


## 交换资源

如果想在发送给服务端的请求中设置头信息的话，可以使用RestTemplate的exchange()方法：

```java

/**
 * Execute the HTTP method to the given URI template, writing the given request entity to the request, and
 * returns the response as {@link ResponseEntity}.
 * <p>URI Template variables are expanded using the given URI variables, if any.
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the request
 * may be {@code null})
 * @param responseType the type of the return value
 * @param uriVariables the variables to expand in the template
 * @return the response as entity
 * @since 3.0.2
 */
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        Class<T> responseType, Object... uriVariables) throws RestClientException;

/**
 * Execute the HTTP method to the given URI template, writing the given request entity to the request, and
 * returns the response as {@link ResponseEntity}.
 * <p>URI Template variables are expanded using the given URI variables, if any.
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the request
 * (may be {@code null})
 * @param responseType the type of the return value
 * @param uriVariables the variables to expand in the template
 * @return the response as entity
 * @since 3.0.2
 */
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        Class<T> responseType, Map<String, ?> uriVariables) throws RestClientException;

/**
 * Execute the HTTP method to the given URI template, writing the given request entity to the request, and
 * returns the response as {@link ResponseEntity}.
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the request
 * (may be {@code null})
 * @param responseType the type of the return value
 * @return the response as entity
 * @since 3.0.2
 */
<T> ResponseEntity<T> exchange(URI url, HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        Class<T> responseType) throws RestClientException;

/**
 * Execute the HTTP method to the given URI template, writing the given
 * request entity to the request, and returns the response as {@link ResponseEntity}.
 * The given {@link ParameterizedTypeReference} is used to pass generic type information:
 * <pre class="code">
 * ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt; myBean =
 *     new ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt;() {};
 *
 * ResponseEntity&lt;List&lt;MyBean&gt;&gt; response =
 *     template.exchange(&quot;http://example.com&quot;,HttpMethod.GET, null, myBean);
 * </pre>
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the
 * request (may be {@code null})
 * @param responseType the type of the return value
 * @param uriVariables the variables to expand in the template
 * @return the response as entity
 * @since 3.2
 */
<T> ResponseEntity<T> exchange(String url,HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        ParameterizedTypeReference<T> responseType, Object... uriVariables) throws RestClientException;

/**
 * Execute the HTTP method to the given URI template, writing the given
 * request entity to the request, and returns the response as {@link ResponseEntity}.
 * The given {@link ParameterizedTypeReference} is used to pass generic type information:
 * <pre class="code">
 * ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt; myBean =
 *     new ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt;() {};
 *
 * ResponseEntity&lt;List&lt;MyBean&gt;&gt; response =
 *     template.exchange(&quot;http://example.com&quot;,HttpMethod.GET, null, myBean);
 * </pre>
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the request
 * (may be {@code null})
 * @param responseType the type of the return value
 * @param uriVariables the variables to expand in the template
 * @return the response as entity
 * @since 3.2
 */
<T> ResponseEntity<T> exchange(String url, HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        ParameterizedTypeReference<T> responseType, Map<String, ?> uriVariables) throws RestClientException;

/**
 * Execute the HTTP method to the given URI template, writing the given
 * request entity to the request, and returns the response as {@link ResponseEntity}.
 * The given {@link ParameterizedTypeReference} is used to pass generic type information:
 * <pre class="code">
 * ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt; myBean =
 *     new ParameterizedTypeReference&lt;List&lt;MyBean&gt;&gt;() {};
 *
 * ResponseEntity&lt;List&lt;MyBean&gt;&gt; response =
 *     template.exchange(&quot;http://example.com&quot;,HttpMethod.GET, null, myBean);
 * </pre>
 * @param url the URL
 * @param method the HTTP method (GET, POST, etc)
 * @param requestEntity the entity (headers and/or body) to write to the request
 * (may be {@code null})
 * @param responseType the type of the return value
 * @return the response as entity
 * @since 3.2
 */
<T> ResponseEntity<T> exchange(URI url, HttpMethod method, @Nullable HttpEntity<?> requestEntity,
        ParameterizedTypeReference<T> responseType) throws RestClientException;

/**
 * Execute the request specified in the given {@link RequestEntity} and return
 * the response as {@link ResponseEntity}. Typically used in combination
 * with the static builder methods on {@code RequestEntity}, for instance:
 * <pre class="code">
 * MyRequest body = ...
 * RequestEntity request = RequestEntity
 *     .post(new URI(&quot;http://example.com/foo&quot;))
 *     .accept(MediaType.APPLICATION_JSON)
 *     .body(body);
 * ResponseEntity&lt;MyResponse&gt; response = template.exchange(request, MyResponse.class);
 * </pre>
 * @param requestEntity the entity to write to the request
 * @param responseType the type of the return value
 * @return the response as entity
 * @since 4.1
 */
<T> ResponseEntity<T> exchange(RequestEntity<?> requestEntity, Class<T> responseType)
        throws RestClientException;

/**
 * Execute the request specified in the given {@link RequestEntity} and return
 * the response as {@link ResponseEntity}. The given
 * {@link ParameterizedTypeReference} is used to pass generic type information:
 * <pre class="code">
 * MyRequest body = ...
 * RequestEntity request = RequestEntity
 *     .post(new URI(&quot;http://example.com/foo&quot;))
 *     .accept(MediaType.APPLICATION_JSON)
 *     .body(body);
 * ParameterizedTypeReference&lt;List&lt;MyResponse&gt;&gt; myBean =
 *     new ParameterizedTypeReference&lt;List&lt;MyResponse&gt;&gt;() {};
 * ResponseEntity&lt;List&lt;MyResponse&gt;&gt; response = template.exchange(request, myBean);
 * </pre>
 * @param requestEntity the entity to write to the request
 * @param responseType the type of the return value
 * @return the response as entity
 * @since 4.1
 */
<T> ResponseEntity<T> exchange(RequestEntity<?> requestEntity, ParameterizedTypeReference<T> responseType)
        throws RestClientException;
        
```

exchange()使用HttpMethod参数来表明要使用的HTTP动作，根据这个参数的值，exchange()能够知晓与其他RestTemplate方法的一样的工作：

```java
public User getRemoteUser(String id) {
    Map<String, String> urlVariables = Collections.singletonMap("id", id);
    RestTemplate rest = new RestTemplate();
    ResponseEntity<User> response = rest.exchange(".../api/user/{userId}", HttpMethod.GET, null, User.class,
            urlVariables);
    return response.getBody();
}
```

使用exchange()在请求头中设置信息：希望服务端以JSON格式发送资源(将Accept设置为‘application/json’)

```java
public User getRemoteUser(String id) {
    Map<String, String> urlVariables = Collections.singletonMap("id", id);
    
    RestTemplate rest = new RestTemplate();
    
    MultiValueMap<String, String> headers=new LinkedMultiValueMap<>(); //创建LinkedMultiValueMap并添加值为“application/json”的Accept头信息
    headers.add("Accept", "application/json");
    //构建HttpEntity将MultiValueMap作为构造参数传入
    ResponseEntity<User> response = rest.exchange(".../api/user/{userId}", HttpMethod.GET, new HttpEntity<>(headers), User.class,
            urlVariables);
    
    return response.getBody();
}
```
如果是一个PUT或者POST请求，需要为HttpEntity设置在请求体中发送的对象

## 总结

RESTful架构使用WEB标准来集成应用程序，使得交互变得简单自然，系统中的资源采用URL进行标识，使用HTTP方法进行管理并且会以一种或多种合适客户端方式来进行表述；借助于RestTemplate编写响应RESTfull资源管理，将参数化的URL模式与特定的HTTP方法关联起来，来处理资源的GET、POST、PUT、DELETE请求


