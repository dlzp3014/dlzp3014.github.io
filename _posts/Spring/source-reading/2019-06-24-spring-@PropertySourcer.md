---
layout: post
title:  "Spring源码-@PropertySource注解"
date:   2019-06-23 22:37:00
categories: Spring 
tags: Spring-Source-Reading Spring-Context
---

* content
{:toc}

@PropertySource注解用于加载指定路径下的属性文件(PropertySource)到Environment中，从而可从Environment对象中获取指定属性的值，如下：

```java
@PropertySource("classpath:config/config.properties")
@Configurable
public class AppConfig {

    @Test
    public void test() {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        CustomConfig customConfig = applicationContext.getBean(CustomConfig.class);
        Assert.assertEquals(customConfig.getName(),"dlzp");
    }

    @Autowired
    Environment environment;

    @Bean
    public CustomConfig customConfig() {
        String name = environment.getProperty("custom.config.name", String.class);
        Integer age = environment.getProperty("custom.config.age", int.class);
        CustomConfig customConfig = new CustomConfig();
        customConfig.setAge(age);
        customConfig.setName(name);
        return customConfig;
    }
}
public class CustomConfig {

    private String name;
    private int age;
    //xxSet/getMethod
}

config/config.properties:

custom.config.name=dlzp
custom.config.age=30
```



## @PropertySource注解定义


```java
/**
 * Annotation providing a convenient and declarative mechanism 声明性机制 for adding a
 * {@link org.springframework.core.env.PropertySource PropertySource 属性资源} to Spring's
 * {@link org.springframework.core.env.Environment Environment}. To be used in
 * conjunction with @{@link Configuration} classes. 与Configuration连接使用
 *
 * <h3>Example usage</h3>
 *
 * <p>Given a file {@code app.properties} containing the key/value pair
 * {@code testbean.name=myTestBean}, the following {@code @Configuration} class
 * uses {@code @PropertySource} to contribute {@code app.properties} to the
 * {@code Environment}'s set of {@code PropertySources}.
 *
 * <pre class="code">
 * &#064;Configuration
 * &#064;PropertySource("classpath:/com/myco/app.properties")
 * public class AppConfig {
 *
 *     &#064;Autowired
 *     Environment env; //自动导入
 *
 *     &#064;Bean
 *     public TestBean testBean() {
 *         TestBean testBean = new TestBean();
 *         testBean.setName(env.getProperty("testbean.name"));
 *         return testBean;
 *     }
 * }</pre>
 *
 * Notice that the {@code Environment} object is
 * {@link org.springframework.beans.factory.annotation.Autowired @Autowired} into the
 * configuration class and then used when populating 填充 the {@code TestBean} object. Given
 * the configuration above, a call to {@code testBean.getName()} will return "myTestBean".
 *
 * <h3>Resolving ${...} placeholders in {@code <bean>} and {@code @Value} annotations</h3>
 *
 * In order to resolve ${...} placeholders in {@code <bean>} definitions or {@code @Value}
 * annotations using properties from a {@code PropertySource}, one must register 
 * a {@code PropertySourcesPlaceholderConfigurer}. This happens automatically when using
 * {@code <context:property-placeholder>} in XML, but must be explicitly 明确地 registered using
 * a {@code static} {@code @Bean} method when using {@code @Configuration} classes. See
 * the "Working with externalized values" section of @{@link Configuration}'s javadoc and
 * "a note on BeanFactoryPostProcessor-returning @Bean methods" of @{@link Bean}'s javadoc
 * for details and examples.
 *
 * <h3>Resolving ${...} placeholders within {@code @PropertySource} resource locations</h3>
 *
 * Any ${...} placeholders present in a {@code @PropertySource} {@linkplain #value() 
 * resource location} will be resolved against the set of property sources already
 * registered against the environment. For example:
 *
 * <pre class="code">
 * &#064;Configuration
 * &#064;PropertySource("classpath:/com/${my.placeholder:default/path}/app.properties")
 * public class AppConfig {
 *
 *     &#064;Autowired
 *     Environment env;
 *
 *     &#064;Bean
 *     public TestBean testBean() {
 *         TestBean testBean = new TestBean();
 *         testBean.setName(env.getProperty("testbean.name"));
 *         return testBean;
 *     }
 * }</pre>
 *
 * Assuming that "my.placeholder" is present in one of the property sources already
 * registered, e.g. system properties or environment variables, the placeholder will
 * be resolved to the corresponding value 对应值. If not, then "default/path" will be used as a
 * default. Expressing a default value (delimited by colon ":") is optional.  If no
 * default is specified and a property cannot be resolved, an {@code
 * IllegalArgumentException} will be thrown.
 *
 * <h3>A note on property overriding with @PropertySource</h3>
 *
 * In cases where a given property key exists in more than one {@code .properties}
 * file, the last {@code @PropertySource} annotation processed will 'win' and override. 
 *
 * For example, given two properties files {@code a.properties} and
 * {@code b.properties}, consider the following two configuration classes
 * that reference them with {@code @PropertySource} annotations:
 *
 * <pre class="code">
 * &#064;Configuration
 * &#064;PropertySource("classpath:/com/myco/a.properties")
 * public class ConfigA { }
 *
 * &#064;Configuration
 * &#064;PropertySource("classpath:/com/myco/b.properties")
 * public class ConfigB { }
 * </pre>
 *
 * The override ordering depends on the order in which these classes are registered
 * with the application context.
 *
 * <pre class="code">
 * AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
 * ctx.register(ConfigA.class);
 * ctx.register(ConfigB.class);
 * ctx.refresh();
 * </pre>
 *
 * In the scenario above 再上面的场景中, the properties in {@code b.properties} will override any
 * duplicates that exist in {@code a.properties}, because {@code ConfigB} was registered 最后注册
 * last.
 *
 * <p>In certain situations 在某些情况下, it may not be possible or practical 实际 to tightly 严格 control
 * property source ordering when using {@code @ProperySource} annotations. For example,
 * if the {@code @Configuration} classes above were registered via component-scanning,
 * the ordering is difficult to predict 顺序很难预测. In such cases - and if overriding is important -
 * it is recommended that the user fall back to using the programmatic PropertySource API.
 * See {@link org.springframework.core.env.ConfigurableEnvironment ConfigurableEnvironment}
 * and {@link org.springframework.core.env.MutablePropertySources MutablePropertySources}
 * javadocs for details.
 *
 * <p><b>NOTE: This annotation is repeatable according to Java 8 conventions.</b>
 * However, all such {@code @PropertySource} annotations need to be declared at the same
 * level: either directly on the configuration class or as meta-annotations within the
 * same custom annotation. Mixing 混合 of direct annotations and meta-annotations is not
 * recommended since direct annotations will effectively override meta-annotations.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @author Phillip Webb
 * @since 3.1
 * @see PropertySources
 * @see Configuration
 * @see org.springframework.core.env.PropertySource
 * @see org.springframework.core.env.ConfigurableEnvironment#getPropertySources()
 * @see org.springframework.core.env.MutablePropertySources
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(PropertySources.class)
public @interface PropertySource {

	/**
	 * Indicate the name of this property source 指示此属性源的名称. If omitted 遗留, a name will
	 * be generated based on the description of the underlying resource. 资源描述
	 * @see org.springframework.core.env.PropertySource#getName()
	 * @see org.springframework.core.io.Resource#getDescription()
	 */
	String name() default "";

	/**
	 * Indicate the resource location(s) of the properties file to be loaded. 资源加载路径
	 * <p>Both traditional 传统 and XML-based properties file formats are supported
	 * &mdash; for example, {@code "classpath:/com/myco/app.properties"}
	 * or {@code "file:/path/to/file.xml"}.
	 * <p>Resource location wildcards (e.g. *&#42;/*.properties) are not permitted; 通配符不允许
	 * each location must evaluate to exactly one {@code .properties} resource.
	 * <p>${...} placeholders will be resolved against any/all property sources already 已经存在在Environment中的属性源
	 * registered with the {@code Environment}. See {@linkplain PropertySource above}
	 * for examples.
	 * <p>Each location will be added to the enclosing 封闭 {@code Environment} as its own
	 * property source, and in the order declared.
	 */
	String[] value();

	/**
	 * Indicate if failure to find the a {@link #value() property resource} should be
	 * ignored. 属于源未找到时忽略
	 * <p>{@code true} is appropriate 适当的if the properties file is completely optional.
	 * Default is {@code false}.
	 * @since 4.0
	 */
	boolean ignoreResourceNotFound() default false;

	/**
	 * A specific character encoding for the given resources, e.g. "UTF-8". 给定资源编码
	 * @since 4.3
	 */
	String encoding() default "";

	/**
	 * Specify a custom {@link PropertySourceFactory}, if any. 定义一个自定义的属性源工厂
	 * <p>By default, a default factory for standard resource files will be used.
	 * @since 4.3
	 * @see org.springframework.core.io.support.DefaultPropertySourceFactory
	 * @see org.springframework.core.io.support.ResourcePropertySource
	 */
	Class<? extends PropertySourceFactory> factory() default PropertySourceFactory.class;

}

```

【Note】:

- `需要解析@Value注解中${}占位符或者@PropertySource指定的路径中有${}时，需要注册PropertySourcesPlaceholderConfigurer`：

```java

@Bean
public static PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer(){

	return new PropertySourcesPlaceholderConfigurer();

}
```

- 加载多个属性文件时后面的属性会覆盖前面相同的key


- 可以使用factory属性方法指定自定义的PropertySourceFactory实现，如加载YML或者JSON文件


## @PropertySource加载Yml配置文件

```java

/**
 * YML属性源加载工厂
 */
public class YmlPropertySourceFactory implements PropertySourceFactory {

    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
        //name empty 
        if (!StringUtils.hasLength(name)) {
            name = resource.getResource().getDescription();
        }

        YamlPropertiesFactoryBean yamlPropertiesFactoryBean = new YamlPropertiesFactoryBean();
        yamlPropertiesFactoryBean.setResources(resource.getResource());
        
        Properties properties = yamlPropertiesFactoryBean.getObject();
        return new PropertiesPropertySource(name, properties);
    }

}


@PropertySource(value = "classpath:config/config.yml", factory = YmlPropertySourceFactory.class)
@Configurable
public class AppConfig {

	...
}

config/config.yml

custom:
    config:
        age: 30
        name: dlzp
```


## @PropertySource加载Json配置文件

```java
/**
 * JSON 属性源加载工厂
 */
public class JsonPropertySourceFactory implements PropertySourceFactory {
    @Override
    public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
        //name empty
        if (!StringUtils.hasLength(name)) {
            name = resource.getResource().getDescription();
        }

        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, Object> map = objectMapper.readValue(resource.getInputStream(), Map.class);
        Map<String, Object> result = new HashMap<>();

        resolveKey("", result, map);
        return new MapPropertySource(name, result);

    }

    /**
     * 递归读取json属性
     *
     * @param prefix
     * @param result
     * @param map
     */
    private void resolveKey(String prefix, Map<String, Object> result, Map<String, Object> map) {
        if (prefix.length() > 0) {
            prefix += ".";
        }
        for (Map.Entry entrySet : map.entrySet()) {
            if (entrySet.getValue() instanceof Map) {
                resolveKey(prefix + entrySet.getKey(), result, (Map<String, Object>) entrySet.getValue());
            } else {
                result.put(prefix + entrySet.getKey().toString(), entrySet.getValue());
            }
        }
    }
}


```


## @PropertySource属性源加载逻辑

由于@PropertySource属性源加载时会使用PropertySourceFactory属性源工厂接口提供的`PropertySource<?> createPropertySource(@Nullable String name, EncodedResource resource) throws IOException;`方法，因此可将断点定位到此处，执行流程如下：

![](/img/post.img/spring/@PropertySource.png)



【说明】：

1.属性源加载在refresh上下文中进行，随后调用Bean工厂的后置处理器立即委托给PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors方法执行(Delegate for AbstractApplicationContext's post-processor handling)

2.由于@PropertySource需要结合@Configuration一起使用，所以得使用ConfigurationClassPostProcessor#processConfigBeanDefinitions方法进行解析

3.随后执行ConfigurationClassPostProcessor#doProcessConfigurationClass方法解析@PropertySource注释、加载属性源、添加到environment环境信息中：`MutablePropertySources propertySources = ((ConfigurableEnvironment) this.environment).getPropertySources()`



代码如下：

```java

protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {

	......

	// Process any @PropertySource annotations 
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable( //配置类中获取@PropertySources注解，JDK8后支持重复注释
			sourceClass.getMetadata(), PropertySources.class,
			org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) { //使用为配置环境信息
			processPropertySource(propertySource);
		}
		else {
			logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}
	......
}


/**
 * Process the given <code>@PropertySource</code> annotation metadata.
 * @param propertySource metadata for the <code>@PropertySource</code> annotation found
 * @throws IOException if loading a property source failed
 */
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
	String name = propertySource.getString("name");
	if (!StringUtils.hasLength(name)) {
		name = null;
	}
	String encoding = propertySource.getString("encoding");
	if (!StringUtils.hasLength(encoding)) {
		encoding = null;
	}
	String[] locations = propertySource.getStringArray("value"); //属性源位置
	Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
	boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound"); //资源未找到时忽略

	Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory"); //属性源加载工厂
	PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
			DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass)); 

	for (String location : locations) {
		try {
			//解析占位符
			String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
			Resource resource = this.resourceLoader.getResource(resolvedLocation); //加载资源
			addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
		}
		catch (IllegalArgumentException | FileNotFoundException | UnknownHostException ex) {
			// Placeholders not resolvable or resource not found when trying to open it
			if (ignoreResourceNotFound) {
				if (logger.isInfoEnabled()) {
					logger.info("Properties location [" + location + "] not resolvable: " + ex.getMessage());
				}
			}
			else {
				throw ex;
			}
		}
	}
}

private void addPropertySource(PropertySource<?> propertySource) {
		String name = propertySource.getName();
		//多属性源对象
		MutablePropertySources propertySources = ((ConfigurableEnvironment) this.environment).getPropertySources();
		//提供属性源的名字是否已包含，包含后缀加
		if (this.propertySourceNames.contains(name)) {
			// We've already added a version, we need to extend it
			PropertySource<?> existing = propertySources.get(name);
			if (existing != null) {
				PropertySource<?> newSource = (propertySource instanceof ResourcePropertySource ?
						((ResourcePropertySource) propertySource).withResourceName() : propertySource);
				if (existing instanceof CompositePropertySource) { //组合
					((CompositePropertySource) existing).addFirstPropertySource(newSource);
				}
				else {
					if (existing instanceof ResourcePropertySource) {
						existing = ((ResourcePropertySource) existing).withResourceName();
					}
					//组合两个ResourcePropertySource
					CompositePropertySource composite = new CompositePropertySource(name);
					composite.addPropertySource(newSource);
					composite.addPropertySource(existing);
					propertySources.replace(name, composite);
				}
				return;
			}
		}
		//添加到最后
		if (this.propertySourceNames.isEmpty()) {
			propertySources.addLast(propertySource);
		}
		else {
			//缀加
			String firstProcessed = this.propertySourceNames.get(this.propertySourceNames.size() - 1);
			propertySources.addBefore(firstProcessed, propertySource);
		}
		this.propertySourceNames.add(name);
	}
```

