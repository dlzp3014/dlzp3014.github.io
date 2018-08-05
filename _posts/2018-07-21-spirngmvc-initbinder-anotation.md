---
layout: post
title:  "SpringMVC使用@InitBinder定制参数转换器"
date:   2018-08-04 14:45:00
categories: SpringMVC 
tags: SpringMVC
---

* content
{:toc}

自定义springMVC的属性编辑器主要有：
- @Controller注解中使用@InitBinder在运行期注册一个属性编辑器，这种编辑器只在当前里面有效
- @ControllerAdvice注解中使用@InitBinder进行全局设置
- 自定义HandlerMethodArgumentResolver 参数解析器，仅使用form表单提交
- 实现Converter接口



使用如下

## 第一种方式 ：仅对当前Controller成有效

```java
@RestController
public class BinderDate {
	private final static Logger LOGGER = LoggerFactory.getLogger(BinderDate.class);

	@GetMapping("bindDate")
	public void bindDate(@RequestParam Date date) {
		LOGGER.info("binder date :{}", date);
	}

	/**
	 * 
	 * Description:使用SpringMvc提供的CustomDateEditor日期编辑器，仅对当前Controller有效
	 * 
	 * @param binder
	 */
	@InitBinder
	public void initBinder(ServletRequestDataBinder binder) {
		SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
		binder.registerCustomEditor(Date.class, new CustomDateEditor(sdf, true)); 
	}

	/**
	 * 
	 * Description:自定义属性编辑器，仅对当前Controller有效
	 * 
	 * @param binder
	 */
	@InitBinder
	public void initBinder(WebDataBinder dataBinder) {
		dataBinder.registerCustomEditor(Date.class, new PropertyEditorSupport() {
			@Override
			public String getAsText() {
				return super.getAsText();
			}

			@Override
			public void setAsText(String text) throws IllegalArgumentException {

				SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
				try {
					setValue(sdf.parse(text));
				} catch (ParseException e) {
					LOGGER.error(e.getMessage());
				}
			}
		});
	}

}
```

使用如下

## 第二种方式 ：配置全局属性编辑器

```java
@ControllerAdvice
public class DateConverter {

	private final static Logger LOGGER = LoggerFactory.getLogger(DateConverter.class);

	@InitBinder
	public void initBinder(DataBinder dataBinder) {

		dataBinder.addCustomFormatter(new Formatter<Date>() {
			DateFormat dateFormat;

			@Override
			public String print(Date object, Locale locale) {
				return dateFormat.format(object);
			}

			@Override
			public Date parse(String text, Locale locale) throws ParseException {
				if (text.matches("^\\d{4}-\\d{1,2}-\\d{1,2}$")) {
					dateFormat = new SimpleDateFormat("yyyy-MM-dd");
				} else {
					dateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
				}
				return dateFormat.parse(text);

			}

		}, Date.class);
	}

}
```

## 第三种方式：实现HandlerMethodArgumentResolver接口

* 自定义转换注解

```java
/**
 * 
 *
 * @ClassName: CustomDateConvert
 * @Description: TODO 针对Form表单提供的自定义类型转换器
 * @author dlzp
 * @date 2018年8月5日 下午6:46:03
 *
 */
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CustomDateConvert {

	/**
	 * 
	 * Description: 需要转换的属性名:注解在Date上仅使用一次
	 * 
	 * @return
	 */
	String[] params() default "";

	String pattern() default "yyyy-MM-dd HH:mm:ss";

}

```

* 自定义HandlerMethodArgumentResolver 参数解析器

```java
public class DateResolver implements HandlerMethodArgumentResolver {

	private String[] params;

	private String pattern;

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		// 是否包含自定义的注解
		if (!parameter.hasParameterAnnotation(CustomDateConvert.class)) {
			return false;
		}
		CustomDateConvert convert = parameter.getParameterAnnotation(CustomDateConvert.class);
		String[] parameters = convert.params();
		if (parameters.length <= 0) {
			return false;
		}
		// 获取注解定义的值
		params = parameters;
		pattern = convert.pattern();
		return true;
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
		HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
		SimpleDateFormat sdf = new SimpleDateFormat(this.pattern);

		if (parameter.getParameterType().getName().equals(Date.class.getName())) {
			return sdf.parse(request.getParameter(params[0]));
		}
		Object object = BeanUtils.instantiateClass(parameter.getParameterType()); // 使用spring提供的BeanUtils根据类型创建当前注解对应的实例

		BeanInfo beanInfo = Introspector.getBeanInfo(object.getClass());

		for (String param : params) {
			String value = request.getParameter(param);
			if (!StringUtils.isEmpty(value)) {
				Date date = sdf.parse(value);
				PropertyDescriptor[] propertyDescriptors = beanInfo.getPropertyDescriptors();
				for (PropertyDescriptor propertyDescriptor : propertyDescriptors) {
					if (propertyDescriptor.getName().equals(param)) {
						Method writeMethod = propertyDescriptor.getWriteMethod();
						if (!Modifier.isPublic(writeMethod.getModifiers())) {
							writeMethod.setAccessible(true);
						}
						writeMethod.invoke(object, date);
					}
				}
			}
		}
		return object;
	}

}

```

* 配置自定义解析器

```java
@Configuration
public class webConfig implements WebMvcConfigurer {
	
	@Override
	public void addArgumentResolvers(List<HandlerMethodArgumentResolver> argumentResolvers) {
		argumentResolvers.add(new DateResolver());
	}
	

}
```
* 测试

```java

@GetMapping("bindDate")
public void bindDate(@CustomDateConvert(params = "date") Date date) {
    LOGGER.info("binder date :{}", date);
}
//bindDate?date=2018-08-21 2016:59:24  
```

## 第四种方式：实现Converter接口

```
@Component
public class StringToDateConverter implements Converter<String, Date> {

	@Override
	public Date convert(String value) {

		if (StringUtils.isEmpty(value)) {
			return null;
		}
		DateFormat dateFormat;
		if (value.matches("^\\d{4}-\\d{1,2}-\\d{1,2}$")) {
			dateFormat = new SimpleDateFormat("yyyy-MM-dd");
		} else {
			dateFormat = new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
		}
		try {
			return dateFormat.parse(value);
		} catch (ParseException e) {
			throw new RuntimeException(String.format("parser %s to Date fail", value));
		}
	}

}
```
