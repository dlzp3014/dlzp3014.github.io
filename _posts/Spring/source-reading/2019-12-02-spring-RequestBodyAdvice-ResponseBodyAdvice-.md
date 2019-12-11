---
layout: post
title:  "Spring源码-RequestBodyAdvice、ResponseBodyAdvice定制请求、响应体通知接口"
date:   2019-12-01 08:38:00
categories: Spring 
tags: Spring-Source-Reading
---

* content
{:toc}

RequestBodyAdvice和ResponseBodyAdvice接口都可以对请求、响应体进行定制化处理，如使用ResponseBodyAdvice接口可以实现统一的响应格式或者使用RequestBodyAdvice可以实现请求体绑定到对象后，针对某个参数进行统一的加密处理





## RequestBodyAdvice接口

允许在请求体被读取并且转换为一个对象之前对其进行定制，也允许在将结果对象作为`@RequestBody`或`HttpEntity`方法参数传递到控制器方法之前对其进行处理，接口的实现可以直接使用`RequestMappingHandlerAdapter`注册或者使用`@ControllerAdvice`注解注释(自动的检索)，其提供的接口方法如下:


```java
public interface RequestBodyAdvice {

	/**首先调用，检查当前拦截器是否可应用
	 * Invoked first to determine if this interceptor applies.
	 * @param methodParameter the method parameter
	 * @param targetType the target type, not necessarily the same as the method 目标类型可能与方法参数类型不一致
	 * parameter type, e.g. for {@code HttpEntity<String>}.
	 * @param converterType the selected converter type
	 * @return whether this interceptor should be invoked or not
	 */
	boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType);

	/** 第二次调用，在请求体读取和转换之前
	 * Invoked second before the request body is read and converted.
	 * @param inputMessage the request
	 * @param parameter the target method parameter
	 * @param targetType the target type, not necessarily the same as the method
	 * parameter type, e.g. for {@code HttpEntity<String>}.
	 * @param converterType the converter used to deserialize the body 用于反序列化请求体的转换器
	 * @return the input request or a new instance, never 从未 {@code null}
	 */
	HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

	/**第三次或最后调用，请求体转换为对象后
	 * Invoked third (and last) after the request body is converted to an Object.
	 * @param body set to the converter Object before the first advice is called 在调用第一个通知之前将其设置为转换器对象
	 * @param inputMessage the request
	 * @param parameter the target method parameter
	 * @param targetType the target type, not necessarily the same as the method
	 * parameter type, e.g. for {@code HttpEntity<String>}.
	 * @param converterType the converter used to deserialize the body
	 * @return the same body or a new instance
	 */
	Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

	/**第二个(最后调用) 请求体为空
	 * Invoked second (and last) if the body is empty.
	 * @param body usually set to {@code null} before the first advice is called
	 * @param inputMessage the request
	 * @param parameter the method parameter
	 * @param targetType the target type, not necessarily the same as the method
	 * parameter type, e.g. for {@code HttpEntity<String>}.
	 * @param converterType the selected converter type
	 * @return the value to use or {@code null} which may then raise an
	 * {@code HttpMessageNotReadableException} if the argument is required.
	 */
	@Nullable
	Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType);


}
```

### RequestBodyAdviceAdapter: Spring提供的默认实现

```java
/**
 * A convenient starting point for implementing
 * {@link org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice
 * ResponseBodyAdvice} with default method implementations.
 *
 * <p>Sub-classes are required to implement {@link #supports} to return true 
 * depending on when the advice applies.当当前通知应用时，依赖于子类需要实现#supports方法返回true
 *
 * @author Rossen Stoyanchev
 * @since 4.2
 */
public abstract class RequestBodyAdviceAdapter implements RequestBodyAdvice {

	/**
	 * The default implementation returns the InputMessage that was passed in.
	 */
	@Override
	public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType)
			throws IOException {

		return inputMessage;
	}

	/**
	 * The default implementation returns the body that was passed in.
	 */
	@Override
	public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

		return body;
	}

	/**
	 * The default implementation returns the body that was passed in.
	 */
	@Override
	@Nullable
	public Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage,
			MethodParameter parameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType) {

		return body;
	}

}
```

### JsonViewRequestBodyAdvice: 支持`Jackson's @JsonView`注解

支持声明在Spring MVC的`@HttpEntity或@RequestBody`注解的方法参数，源代码如下: 

```java
/**
 * A {@link RequestBodyAdvice} implementation that adds support for Jackson's
 * {@code @JsonView} annotation declared on 声明在 a Spring MVC {@code @HttpEntity} 
 * or {@code @RequestBody} method parameter.
 * 在注释中指定的反序列化视图将传给MappingJackson2HttpMessageConverter，然后反序列化请求体
 * <p>The deserialization view specified in the annotation will be passed in to the
 * {@link org.springframework.http.converter.json.MappingJackson2HttpMessageConverter}
 * which will then use it to deserialize the request body with.
 *尽管@JsonView运行在类上多次指定，
 * <p>Note that despite {@code @JsonView} allowing for more than one class to
 * be specified, the use for a request body advice is only supported with
 * exactly one class argument 只有一个类参数. Consider the use of a composite interface. 考虑复合接口的使用
 *
 * @author Sebastien Deleuze
 * @since 4.2
 * @see com.fasterxml.jackson.annotation.JsonView
 * @see com.fasterxml.jackson.databind.ObjectMapper#readerWithView(Class)
 */
public class JsonViewRequestBodyAdvice extends RequestBodyAdviceAdapter {

	@Override
	public boolean supports(MethodParameter methodParameter, Type targetType,
			Class<? extends HttpMessageConverter<?>> converterType) {
		//HttpMessageConverter类型转换为Jackson2，且方法参数有@JsonView注解
		return (AbstractJackson2HttpMessageConverter.class.isAssignableFrom(converterType) &&
				methodParameter.getParameterAnnotation(JsonView.class) != null);
	}

	@Override
	public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter methodParameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> selectedConverterType) throws IOException {

		JsonView ann = methodParameter.getParameterAnnotation(JsonView.class); 获取JsonView注解
		Assert.state(ann != null, "No JsonView annotation");

		Class<?>[] classes = ann.value(); //注解指定的类
		if (classes.length != 1) { //参数检查
			throw new IllegalArgumentException(
					"@JsonView only supported for request body advice with exactly 1 class argument: " + methodParameter);
		}

		return new MappingJacksonInputMessage(inputMessage.getBody(), inputMessage.getHeaders(), classes[0]);
	}

}
```

使用示例如下: 当添加系统用户时禁止绑定`主键`

请求Body为`{"sysUserId":1,"sysUserName":"dlzp"}`

```java

@PostMapping("sysUserAdd")
public SysUser jsonView(@JsonView(SysUser.SysUserAdd.class) @RequestBody SysUser sysUser) {

    return sysUser; //并未将请求体中的sysUserId绑定到对象中
}
class SysUser {
    private int sysUserId;
    
    @JsonView(SysUserAdd.class)
    private String sysUserName;
}
```


### 自定义RequestBodyAdvice接口实现

请求体绑定完成之后，针对特定的属性进行加密处理

```java
//自动加密注解，目前仅作用于属性上，注解在方法中需自行扩展
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoEncrypt {
}

//自动加密
@ControllerAdvice
public class AutoEncryptRequestBodyAdvice extends RequestBodyAdviceAdapter {

    @Override
    public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

        if (targetType instanceof Class) {
            for (Field field : ((Class) targetType).getDeclaredFields()) {
                if (Objects.nonNull(field.getAnnotation(AutoEncrypt.class))) {
                    return true;
                }
            }
        }

        return false;
    }

    @Override
    public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

        for (Field field : ((Class) targetType).getDeclaredFields()) {
            if (Objects.nonNull(field.getAnnotation(AutoEncrypt.class))) {
                try {
                    field.setAccessible(true);
                    String digest = DigestUtils.md5DigestAsHex(field.get(body).toString().getBytes());
                    field.set(body, digest);
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }

        return body;
    }
}

public class SysUser {


    private int sysUserId;

    @JsonView(SysUserAdd.class)
    private String sysUserName;

    @AutoEncrypt //自动加密注解
    private String password;

}
//测试
@SpringBootApplication
@RestController
public class CustomMain {

    public static void main(String[] args) {
        SpringApplication.run(CustomMain.class);
    }


    @PostMapping("autoEncrypt")
    public SysUser autoEncrypt(@RequestBody SysUser sysUser) {

        return sysUser; 
    }
}


```


## ResponseBodyAdvice<T>接口

允许在执行`@ResponseBody`或`ResponseEntity`控制器方法后，使用`HttpMessageConverter`Http消息转换之前定制响应，接口的实现可以直接使用`RequestMappingHandlerAdapter`和`ExceptionHandlerExceptionResolver`注册，或者使用`@ControllerAdvice`注解注释(被自动的检索到)，其提供的接口方法如下:

```java
//T为返回的类型
public interface ResponseBodyAdvice<T> {

	/**该组件是否支持给定的控制器方法返回类型和已选择的HttpMessageConverter类型，返回true时beforeBodyWrite方法将被调用
	 * Whether this component supports the given controller method return type
	 * and the selected {@code HttpMessageConverter} type.
	 * @param returnType the return type
	 * @param converterType the selected converter type
	 * @return {@code true} if {@link #beforeBodyWrite} should be invoked; 
	 * {@code false} otherwise
	 */
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);

	/**已选择的HttpMessageConverter之后调用，调用其write方法之前
	 * Invoked after an {@code HttpMessageConverter} is selected and just before
	 * its write method is invoked.
	 * @param body the body to be written
	 * @param returnType the return type of the controller method 控制器方法返回的类型
	 * @param selectedContentType the content type selected through content negotiation
	 * @param selectedConverterType the converter type selected to write to the response
	 * @param request the current request
	 * @param response the current response
	 * @return the body that was passed in or a modified (possibly new) instance
	 */
	@Nullable
	T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType,
			Class<? extends HttpMessageConverter<?>> selectedConverterType,
			ServerHttpRequest request, ServerHttpResponse response);

}

```


### AbstractMappingJacksonResponseBodyAdvice: Jackson序列化Advice基类

```java
/**
 * A convenient base class for {@code ResponseBodyAdvice} implementations
 * that customize the response 自定义响应 before JSON serialization with
 * {@link AbstractJackson2HttpMessageConverter}'s concrete subclasses. 具体的子类
 *
 * @author Rossen Stoyanchev
 * @author Sebastien Deleuze
 * @since 4.1
 */
public abstract class AbstractMappingJacksonResponseBodyAdvice implements ResponseBodyAdvice<Object> {

	@Override
	public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
		return AbstractJackson2HttpMessageConverter.class.isAssignableFrom(converterType);
	}

	//响应体写出前执行
	@Override 
	@Nullable
	public final Object beforeBodyWrite(@Nullable Object body, MethodParameter returnType,
			MediaType contentType, Class<? extends HttpMessageConverter<?>> converterType,
			ServerHttpRequest request, ServerHttpResponse response) {

		if (body == null) {
			return null;
		}
		MappingJacksonValue container = getOrCreateContainer(body); //Jackson值映射
		beforeBodyWriteInternal(container, contentType, returnType, request, response);
		return container;
	}

	/**
	 * Wrap the body in a {@link MappingJacksonValue} value container (for providing
	 * additional serialization instructions 序列化指定) or simply cast it if already wrapped.
	 */
	protected MappingJacksonValue getOrCreateContainer(Object body) {
		return (body instanceof MappingJacksonValue ? (MappingJacksonValue) body : new MappingJacksonValue(body));
	}

	/**
	 * Invoked only if the converter type is {@code MappingJackson2HttpMessageConverter}.
	 */
	protected abstract void beforeBodyWriteInternal(MappingJacksonValue bodyContainer, MediaType contentType,
			MethodParameter returnType, ServerHttpRequest request, ServerHttpResponse response);

}
```


### JsonViewResponseBodyAdvice: 支持`Jackson's @JsonView`注解

```java
/**
 * A {@link ResponseBodyAdvice} implementation that adds support for Jackson's
 * {@code @JsonView} annotation declared on a Spring MVC {@code @RequestMapping}
 * or {@code @ExceptionHandler} method.
 *
 * <p>The serialization view specified in the annotation will be passed in to the
 * {@link org.springframework.http.converter.json.MappingJackson2HttpMessageConverter}
 * which will then use it to serialize the response body.
 *
 * <p>Note that despite {@code @JsonView} allowing for more than one class to
 * be specified, the use for a response body advice is only supported with
 * exactly one class argument. Consider the use of a composite interface.
 *
 * @author Rossen Stoyanchev
 * @since 4.1
 * @see com.fasterxml.jackson.annotation.JsonView
 * @see com.fasterxml.jackson.databind.ObjectMapper#writerWithView(Class)
 */
public class JsonViewResponseBodyAdvice extends AbstractMappingJacksonResponseBodyAdvice {

	@Override
	public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
		return super.supports(returnType, converterType) && returnType.hasMethodAnnotation(JsonView.class); //返回类型有有@JsonView注解
	}

	@Override
	protected void beforeBodyWriteInternal(MappingJacksonValue bodyContainer, MediaType contentType,
			MethodParameter returnType, ServerHttpRequest request, ServerHttpResponse response) {

		JsonView ann = returnType.getMethodAnnotation(JsonView.class);
		Assert.state(ann != null, "No JsonView annotation");

		Class<?>[] classes = ann.value();
		if (classes.length != 1) {
			throw new IllegalArgumentException(
					"@JsonView only supported for response body advice with exactly 1 class argument: " + returnType);
		}

		bodyContainer.setSerializationView(classes[0]);  //设置视图类型
	}

}
```

使用示例如下: 返回系统用户时禁止响应`密码`

```java
public class SysUser {

    @JsonView(SysUserInfo.class)
    private int sysUserId;

    @JsonView({SysUserInfo.class})
    private String sysUserName;

    public interface SysUserInfo {
    }
}
@GetMapping("sysUser")
@JsonView(SysUser.SysUserInfo.class)
public SysUser sysUserInfo() {

    SysUser sysUser = new SysUser();
    sysUser.setSysUserId(1);
    sysUser.setSysUserName("dlzp");
    sysUser.setPassword("123456");
    return sysUser;
}


```

### 自定义ResponseBodyAdvice<T>接口实现

统一返回响应体: 基于AbstractJackson2HttpMessageConverter，且使用自定义注解实现

```java
//统一格式化注解
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AutoFormatResponse {
}

//统一响应格式
public class FormateResponse {

    private Integer code;

    private String msg;

    private Object data;

    public static FormateResponse success() {
        FormateResponse response = new FormateResponse();
        response.setCode(1);
        response.setMsg("success");
        return response;
    }

    public static FormateResponse success(Object data) {
        FormateResponse response = success();
        response.setData(data);
        return response;
    }

    ...
}

//统一响应通知:基于Jackson
@ControllerAdvice
public class FormatResponseBodyAdvice extends AbstractMappingJacksonResponseBodyAdvice {

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return super.supports(returnType, converterType) && Objects.nonNull(returnType.getMethodAnnotation(AutoFormatResponse.class));
    }

    @Override
    protected void beforeBodyWriteInternal(MappingJacksonValue bodyContainer, MediaType contentType, MethodParameter returnType, ServerHttpRequest request, ServerHttpResponse response) {
        Object before = bodyContainer.getValue();
        if (!(before instanceof FormateResponse)) {
            bodyContainer.setValue(FormateResponse.success(bodyContainer.getValue()));
        }
    }
}

//测试

@GetMapping("sysUser")
@AutoFormatResponse
public SysUser sysUserInfo() {

    SysUser sysUser = new SysUser();
    sysUser.setSysUserId(1);
    sysUser.setSysUserName("dlzp");
    sysUser.setPassword("123456");
    return sysUser;
}
{
    "code": 1,
    "msg": "success",
    "data": {
        "sysUserId": 1,
        "sysUserName": "dlzp",
        "password": "123456"
    }
}
```



## RequestResponseBodyAdviceChain: 请求响应体通知链

Spring框架中的包级私有，同时实现RequestBodyAdvice和ResponseBodyAdvice接口，内部组合了多个RequestBodyAdvice和ResponseBodyAdvice接口的实现，并将所有的实现进行链式调用

```java
/**
 * Invokes {@link RequestBodyAdvice} and {@link ResponseBodyAdvice} where each
 * instance may be (and is most likely) wrapped with 可能被包装为一个ControllerAdviceBean实例(使用@ControllerAdvice注解进行注释其接口的实现)
 * {@link org.springframework.web.method.ControllerAdviceBean ControllerAdviceBean}.
 *
 * @author Rossen Stoyanchev
 * @since 4.2
 */
class RequestResponseBodyAdviceChain implements RequestBodyAdvice, ResponseBodyAdvice<Object> {

	private final List<Object> requestBodyAdvice = new ArrayList<>(4); //RequestBodyAdvice列表

	private final List<Object> responseBodyAdvice = new ArrayList<>(4);//ResponseBodyAdvice列表


	/**
	 * Create an instance from a list of objects that are either of type 从列表对象中创建实例，列表类型为ControllerAdviceBean或者RequestBodyAdvice
	 * {@code ControllerAdviceBean} or {@code RequestBodyAdvice}.
	 */
	public RequestResponseBodyAdviceChain(@Nullable List<Object> requestResponseBodyAdvice) {
		if (requestResponseBodyAdvice != null) {
			for (Object advice : requestResponseBodyAdvice) {
				//为ControllerAdviceBean对象(获取Bean类型)，否则获取通知类型
				Class<?> beanType = (advice instanceof ControllerAdviceBean ?
						((ControllerAdviceBean) advice).getBeanType() : advice.getClass());
				if (beanType != null) {
					if (RequestBodyAdvice.class.isAssignableFrom(beanType)) { //实现requestBodyAdvice接口
						this.requestBodyAdvice.add(advice);
					}
					else if (ResponseBodyAdvice.class.isAssignableFrom(beanType)) {//实现ResponseBodyAdvice接口
						this.responseBodyAdvice.add(advice);
					}
				}
			}
		}
	}

	//不支持当前操作
	@Override
	public boolean supports(MethodParameter param, Type type, Class<? extends HttpMessageConverter<?>> converterType) {
		throw new UnsupportedOperationException("Not implemented");
	}

	//不支持当前操作
	@Override
	public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
		throw new UnsupportedOperationException("Not implemented");
	}

	//从HttpInputMessage读取请求体之前
	@Override
	public HttpInputMessage beforeBodyRead(HttpInputMessage request, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException {
		//获取匹配的RequestBodyAdvice通知，遍历执行支持对象#beforeBodyRead方法，
		for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
			if (advice.supports(parameter, targetType, converterType)) {
				request = advice.beforeBodyRead(request, parameter, targetType, converterType);
			}
		}
		return request;
	}

	//从HttpInputMessage读取请求体绑定为body对象后
	@Override
	public Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

		for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
			if (advice.supports(parameter, targetType, converterType)) {
				body = advice.afterBodyRead(body, inputMessage, parameter, targetType, converterType);
			}
		}
		return body;
	}

	//将body对象输出之前调用
	@Override
	@Nullable
	public Object beforeBodyWrite(@Nullable Object body, MethodParameter returnType, MediaType contentType,
			Class<? extends HttpMessageConverter<?>> converterType,
			ServerHttpRequest request, ServerHttpResponse response) {

		return processBody(body, returnType, contentType, converterType, request, response);
	}

	@Override
	@Nullable
	public Object handleEmptyBody(@Nullable Object body, HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {

		for (RequestBodyAdvice advice : getMatchingAdvice(parameter, RequestBodyAdvice.class)) {
			if (advice.supports(parameter, targetType, converterType)) {
				body = advice.handleEmptyBody(body, inputMessage, parameter, targetType, converterType);
			}
		}
		return body;
	}


	@SuppressWarnings("unchecked")
	@Nullable
	private <T> Object processBody(@Nullable Object body, MethodParameter returnType, MediaType contentType,
			Class<? extends HttpMessageConverter<?>> converterType,
			ServerHttpRequest request, ServerHttpResponse response) {
		//获取匹配的ResponseBodyAdvice
		for (ResponseBodyAdvice<?> advice : getMatchingAdvice(returnType, ResponseBodyAdvice.class)) {
			if (advice.supports(returnType, converterType)) {
				body = ((ResponseBodyAdvice<T>) advice).beforeBodyWrite((T) body, returnType,
						contentType, converterType, request, response);
			}
		}
		return body;
	}

	//根据类型获取匹配的advice列表
	@SuppressWarnings("unchecked")
	private <A> List<A> getMatchingAdvice(MethodParameter parameter, Class<? extends A> adviceType) {
		List<Object> availableAdvice = getAdvice(adviceType);
		if (CollectionUtils.isEmpty(availableAdvice)) {
			return Collections.emptyList();
		}
		List<A> result = new ArrayList<>(availableAdvice.size());
		for (Object advice : availableAdvice) {
			if (advice instanceof ControllerAdviceBean) {
				ControllerAdviceBean adviceBean = (ControllerAdviceBean) advice;
				if (!adviceBean.isApplicableToBeanType(parameter.getContainingClass())) { //检查给定的bean类型是@ControllerAdvice注解的实例
					continue;
				}
				advice = adviceBean.resolveBean();//解析
			}
			if (adviceType.isAssignableFrom(advice.getClass())) {
				result.add((A) advice);
			}
		}
		return result;
	}

	//根据类型获取Advice列表
	private List<Object> getAdvice(Class<?> adviceType) {
		if (RequestBodyAdvice.class == adviceType) {
			return this.requestBodyAdvice;
		}
		else if (ResponseBodyAdvice.class == adviceType) {
			return this.responseBodyAdvice;
		}
		else {
			throw new IllegalArgumentException("Unexpected adviceType: " + adviceType);
		}
	}

}
```