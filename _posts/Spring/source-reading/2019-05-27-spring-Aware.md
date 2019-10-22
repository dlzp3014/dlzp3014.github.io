---
layout: post
title:  "Spring源码-Aware接口"
date:   2019-05-27 21:36:00
categories: Spring 
tags: Spring-Source-Reading Spring-Beans
---

* content
{:toc}


XXXAware在Spring里表示对XXX可感知的，通俗解释就是：如果在某个类里面想要使用Spring的一些东西，就可以通过实现XXXAware接口告诉Spring，Spring看到后就会传过来，而接收的方式是通过实现接口接口唯一的方法setXXX。比如有一个类想要使用当前的ApplicationContext，那么只需要让它实现ApplicationContextAware接口，然后实现接口中唯一的方法void setApplicationContext(ApplicationContext applicationContext)就可以，Spring会自动调用这个方法将applicationContext传递过来。如下使用实例：使用ApplicationContextAware(其他AWare子接口使用过程相同)接口实现在`spring环境中获取非spring容器管理的Bean`：

```java
@Component
public class ApplicationContextHolder implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        ApplicationContextHolder.applicationContext = applicationContext;
    }


    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```


Spring框架中提个的AWare接口如下：

- ApplicationContextAware: 应用上下文感知接口
- ApplicationEventPublisherAware: 应用事件发布感知接口
- BeanClassLoaderAware: Bean加载器感知接口
- BeanFactoryAware： Bean工厂感知接口
- BeanNameAware：BeanName感知接口
- EmbeddedValueResolverAware： 内置字符串值解析器感知接口
- EnvironmentAware： 环境感知接口
- ImportAware： 导入配置文件感知接口
- LoadTimeWeaverAware： 类加载期织入感知接口
- MessageSourceAware: 消息源感知接口
- NotificationPublisherAware: JMX notifications感知接口
- ResourceLoaderAware： 资源加载器感知接口
- ServletConfigAware： Servlet配置感知接口
- ServletContextAware： Servlet上下文感知接口




## Aware

```java
/**
 * A marker superinterface indicating标记接口表示 that a bean is eligible to be notified by the 一个bean有资格得到通知
 * Spring container of a particular framework object through a callback-style method. 根据特定框架对象的Spring容器回调样式方法
 * The actual method signature is determined by individual subinterfaces but should 实际方法签名根据独特的子接口确定
 * typically consist of just one void-returning method that accepts a single argument. 
 * 通常由一个接受单个参数的void返回方法组成
 * <p>Note that merely implementing {@link Aware} provides no default functionality. 仅仅实现Aware没有默认的功能
 * Rather, processing must be done explicitly 处理必须显式地完成, for example in a
 * {@link org.springframework.beans.factory.config.BeanPostProcessor}.
 * Refer to {@link org.springframework.context.support.ApplicationContextAwareProcessor}
 * for an example of processing specific {@code *Aware} interface callbacks.
 *
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 */
public interface Aware {

}
```

## ApplicationContextAware

```java
/**
 * Interface to be implemented by any object 接口可以被任何对象实现that wishes to be notified 希望得到运行的ApplicationContext通知
 * of the {@link ApplicationContext} that it runs in.
 *
 * <p>Implementing this interface makes sense 意义 for example when an object
 * requires access to a set of collaborating 协作 beans. Note that configuration 配置变量bean引用更可取
 * via bean references is preferable to implementing this interface just
 * for bean lookup purposes. 用于bean查找
 *
 * <p>This interface can also be implemented if an object needs access to file 访问文件资源
 * resources, i.e. wants to call {@code getResource}, wants to publish 发布应用事件 
 * an application event, or requires access to the MessageSource. However, 访问消息资源
 * it is preferable to implement the more specific {@link ResourceLoaderAware},
 * {@link ApplicationEventPublisherAware} or {@link MessageSourceAware} interface
 * in such a specific scenario  方案.
 *
 * <p>Note that file resource dependencies can also be exposed as bean properties 文件资源的依赖性也能暴露Bean属性的资源类型
 * of type {@link org.springframework.core.io.Resource}, populated via Strings 通过填充字符串
 * with automatic type conversion by the bean factory. This removes the need
 * for implementing any callback interface just for the purpose of accessing
 * a specific file resource.
 *
 * <p>{@link org.springframework.context.support.ApplicationObjectSupport} is a ApplicationObjectSupport是一个方便的基础类
 * convenience base class for application objects, implementing this interface. 用于应用对象，实现这个借口
 *
 * <p>For a list of all bean lifecycle methods, see the
 * {@link org.springframework.beans.factory.BeanFactory BeanFactory javadocs}.
 *
 * @see ResourceLoaderAware
 * @see ApplicationEventPublisherAware
 * @see MessageSourceAware
 * @see org.springframework.context.support.ApplicationObjectSupport
 * @see org.springframework.beans.factory.BeanFactoryAware
 */
public interface ApplicationContextAware extends Aware {

	/**
	 * Set the ApplicationContext that this object runs in. 在运行时ApplicationContext
	 * Normally this call will be used to initialize the object. 通常，这个调用将用于初始化对象
	 * <p>Invoked after population of normal bean properties but before an init callback such
	 	在填充普通bean属性之后调用，但在init回调之前调用
	 * as {@link org.springframework.beans.factory.InitializingBean#afterPropertiesSet()} 
	 * or a custom init-method. Invoked after {@link ResourceLoaderAware#setResourceLoader}, 
	 * {@link ApplicationEventPublisherAware#setApplicationEventPublisher} and
	 * {@link MessageSourceAware}, if applicable.
	 * @param applicationContext the ApplicationContext object to be used by this object
	 * @throws ApplicationContextException in case of context initialization errors
	 * @throws BeansException if thrown by application context methods
	 * @see org.springframework.beans.factory.BeanInitializationException
	 */
	void setApplicationContext(ApplicationContext applicationContext) throws BeansException;

}
```


## ApplicationEventPublisherAware

```java
/**
 * 接口将由希望得到ApplicationEventPublisher通知的任何对象实现
 * Interface to be implemented by any object that wishes to be notified
 * of the ApplicationEventPublisher (typically the ApplicationContext)
 * that it runs in.
 *
 * @see ApplicationContextAware
 */
public interface ApplicationEventPublisherAware extends Aware {

	/**
	 * Set the ApplicationEventPublisher that this object runs in. 设置此对象运行的ApplicationEventPublisher
	 * <p>Invoked after population of normal bean properties (在填充普通bean属性后调用)but before an init
	 * callback(在初始化之前调用) like InitializingBean's afterPropertiesSet or a custom init-method.
	 * Invoked before ApplicationContextAware's setApplicationContext.(在applicationcontext的setApplicationContext之前调用)
	 * @param applicationEventPublisher event publisher to be used by this object
	 */
	void setApplicationEventPublisher(ApplicationEventPublisher applicationEventPublisher);

}
```

## BeanClassLoaderAware

```java
/**
 * Callback that allows a bean to be aware of 感知 the bean
 * {@link ClassLoader class loader}; that is, the class loader used by the
 * present bean factory to load bean classes. 类加载通过存在的bean工厂加载bean类被使用
 *
 * <p>This is mainly intended主要目的 to be implemented by framework classes which
 * have to pick up获得 application classes by name despite尽管  themselves potentially 可能
 * being loaded from a shared class loader.
 *
 * <p>For a list of all bean lifecycle methods, see the
 * {@link BeanFactory BeanFactory javadocs}.
 *
 * @see BeanNameAware
 * @see BeanFactoryAware
 * @see InitializingBean
 */
public interface BeanClassLoaderAware extends Aware {

	/**
	 * Callback that supplies the bean {@link ClassLoader class loader} to
	 * a bean instance. 提供一个ClassLoader的实例
	 * <p>Invoked <i>after</i> the population of normal bean properties but
	 * <i>before</i> an initialization callback such as
	 * {@link InitializingBean InitializingBean's}
	 * {@link InitializingBean#afterPropertiesSet()}
	 * method or a custom init-method.
	 * @param classLoader the owning class loader
	 */
	void setBeanClassLoader(ClassLoader classLoader);

}
```

## BeanFactoryAware

```java
/**
 * Interface to be implemented by beans that wish to be aware of their
 * owning {@link BeanFactory}.
 *
 * <p>For example, beans can look up collaborating合作 beans via the factory
 * (Dependency Lookup). Note that most beans will choose to receive references
 * to collaborating beans via corresponding相应 bean properties or constructor
 * arguments (Dependency Injection).
 *
 * <p>For a list of all bean lifecycle methods, see the
 * {@link BeanFactory BeanFactory javadocs}.
 * @see BeanNameAware
 * @see BeanClassLoaderAware
 * @see InitializingBean
 * @see org.springframework.context.ApplicationContextAware
 */
public interface BeanFactoryAware extends Aware {

	/**
	 * Callback that supplies the owning factory to a bean instance.
	 * <p>Invoked after the population of normal bean properties
	 * but before an initialization callback such as
	 * {@link InitializingBean#afterPropertiesSet()} or a custom init-method.
	 * @param beanFactory owning BeanFactory (never {@code null}).
	 * The bean can immediately call methods on the factory.
	 * @throws BeansException in case of initialization errors
	 * @see BeanInitializationException
	 */
	void setBeanFactory(BeanFactory beanFactory) throws BeansException;

}
```

## BeanNameAware

```java
/**
 * Interface to be implemented by beans that want to be aware of their bean工厂中的bean名字
 * bean name in a bean factory. Note that it is not usually recommended
 * that an object depends on its bean name 对象取决于beanName, as this represents a potentially
 * brittle dependence on external configuration 外部配置, as well as a possibly
 * unnecessary dependence on a Spring API.
 *
 * <p>For a list of all bean lifecycle methods, see the
 * {@link BeanFactory BeanFactory javadocs}.
 *
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 01.11.2003
 * @see BeanClassLoaderAware
 * @see BeanFactoryAware
 * @see InitializingBean
 */
public interface BeanNameAware extends Aware {

	/**
	 * Set the name of the bean in the bean factory that created this bean. 在创建Bean时在Bean工厂中设置Bean的名字
	 * <p>Invoked after population of normal bean properties but before an
	 * init callback such as {@link InitializingBean#afterPropertiesSet()}
	 * or a custom init-method.
	 * @param name the name of the bean in the factory.
	 * Note that this name is the actual 真实 bean name used in the factory, which may 与最初指定的名称不同
	 * differ from the originally specified name: in particular for尤其对于 inner bean
	 * names, the actual bean name might have been made unique through appending 附加后缀
	 * "#..." suffixes. Use the {@link BeanFactoryUtils#originalBeanName(String)}
	 * method to extract the original bean name 提取原始Bean名 (without suffix), if desired.
	 */
	void setBeanName(String name);

}

```

## EmbeddedValueResolverAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified of a
 * <b>StringValueResolver</b> for the <b> resolution of embedded definition values. 
 *
 * <p>This is an alternative另一种选择 to a full ConfigurableBeanFactory dependency via the
 * ApplicationContextAware/BeanFactoryAware interfaces.
 *
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 3.0.3
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#resolveEmbeddedValue(String)
 * @see org.springframework.beans.factory.config.ConfigurableBeanFactory#getBeanExpressionResolver()
 * @see org.springframework.beans.factory.config.EmbeddedValueResolver
 */
public interface EmbeddedValueResolverAware extends Aware {

	/**
	 * Set the StringValueResolver to use for resolving embedded definition values.
	 */
	void setEmbeddedValueResolver(StringValueResolver resolver);

}
```

## EnvironmentAware

```java
/**
 * Interface to be implemented by any bean that wishes to be notified
 * of the {@link Environment} that it runs in.
 *
 * @author Chris Beams
 * @since 3.1
 * @see org.springframework.core.env.EnvironmentCapable
 */
public interface EnvironmentAware extends Aware {

	/**
	 * Set the {@code Environment} that this component runs in. 设置此组件运行的环境
	 */
	void setEnvironment(Environment environment);

}
```

## ImportAware

```java

/** 导入@Configuration类
 * Interface to be implemented by any @{@link Configuration} class that wishes
 * to be injected with the {@link AnnotationMetadata} of the @{@code Configuration}
 * class that imported it. Useful in conjunction with annotations that 结合@Improt作为元注解非常有用
 * use @{@link Import} as a meta-annotation.
 *
 * @author Chris Beams
 * @since 3.1
 */
public interface ImportAware extends Aware {

	/**
	 * Set the annotation metadata of the importing @{@code Configuration} class.
	 */
	void setImportMetadata(AnnotationMetadata importMetadata);

}
```

## LoadTimeWeaverAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified
 * of the application context's default {@link LoadTimeWeaver}. 加载时植入
 *
 * @see org.springframework.context.ConfigurableApplicationContext#LOAD_TIME_WEAVER_BEAN_NAME
 */
public interface LoadTimeWeaverAware extends Aware {

	/**
	 * Set the {@link LoadTimeWeaver} of this object's containing
	 * {@link org.springframework.context.ApplicationContext ApplicationContext}.
	 * <p>Invoked after the population of normal bean properties but before an
	 * initialization callback like
	 * {@link org.springframework.beans.factory.InitializingBean InitializingBean's}
	 * {@link org.springframework.beans.factory.InitializingBean#afterPropertiesSet() afterPropertiesSet()}
	 * or a custom init-method. Invoked after
	 * {@link org.springframework.context.ApplicationContextAware ApplicationContextAware's}
	 * {@link org.springframework.context.ApplicationContextAware#setApplicationContext setApplicationContext(..)}.
	 * <p><b>NOTE:</b> This method will only be called if there actually is a
	 * {@code LoadTimeWeaver} available in the application context. If
	 * there is none, the method will simply not get invoked, assuming that the
	 * implementing object is able to activate its weaving dependency accordingly.
	 * @param loadTimeWeaver the {@code LoadTimeWeaver} instance (never {@code null})
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
	 */
	void setLoadTimeWeaver(LoadTimeWeaver loadTimeWeaver);

}

```

## MessageSourceAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified
 * of the MessageSource (typically the ApplicationContext) that it runs in.
 *
 * <p>Note that the MessageSource can usually also be passed on as bean MessageSource通过也能作为bean引用传递参数
 * reference (to arbitrary bean properties or constructor arguments), because
 * it is defined as bean with name "messageSource" in the application context. 
 *
 * @see ApplicationContextAware
 */
public interface MessageSourceAware extends Aware {

	/**
	 * Set the MessageSource that this object runs in.
	 * <p>Invoked after population of normal bean properties but before an init
	 * callback like InitializingBean's afterPropertiesSet or a custom init-method.
	 * Invoked before ApplicationContextAware's setApplicationContext.
	 * @param messageSource message sourceto be used by this object
	 */
	void setMessageSource(MessageSource messageSource);

}
```

## NotificationPublisherAware

```java
/**
 * Interface to be implemented by any Spring-managed resource that is to be
 * registered with an {@link javax.management.MBeanServer} and wishes to send
 * JMX {@link javax.management.Notification javax.management.Notifications}.
 * 使用NotificationPublisher提个了Spring创建管理资源
 * <p>Provides Spring-created managed resources with a {@link NotificationPublisher}
 * as soon as they are registered with the {@link javax.management.MBeanServer}.
 *
 * <p><b>NOTE:</b> This interface only applies to simple Spring-managed 只用在简单的spring管理的bean,
 * beans which happen to get exported through Spring's
 * {@link org.springframework.jmx.export.MBeanExporter}.
 * It does not apply to any non-exported beans; neither does it apply
 * to standard MBeans exported by Spring. For standard JMX MBeans,
 * consider implementing the {@link javax.management.modelmbean.ModelMBeanNotificationBroadcaster}
 * interface (or implementing a full {@link javax.management.modelmbean.ModelMBean}).
 *
 * @author Rob Harrop
 * @author Chris Beams
 * @since 2.0
 * @see NotificationPublisher
 */
public interface NotificationPublisherAware extends Aware {

	/**
	 * Set the {@link NotificationPublisher} instance for the current managed resource instance.
	 */
	void setNotificationPublisher(NotificationPublisher notificationPublisher);

}
```

## ResourceLoaderAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified of
 * the <b>ResourceLoader</b> (typically the ApplicationContext) that it runs in.
 * This is an alternative to a full ApplicationContext dependency via the
 * ApplicationContextAware interface.
 *
 * <p>Note that Resource dependencies资源依赖 can also be exposed as bean properties 作为Bean属性暴露
 * of type Resource, populated via Strings with automatic type conversion by 通过bean工厂，自动子类转换填充，这样就不需要实现任何回调接口，仅仅是服务指定的文件资源
 * the bean factory. This removes the need for implementing any callback
 * interface just for the purpose of accessing a specific file resource.
 *
 * <p>You typically need a ResourceLoader when your application object has
 * to access a variety of file resources(各种文件资源) whose names are calculated. A good
 * strategy is to make the object use a DefaultResourceLoader but still 一个好的策略是让对象使用DefaultResourceLoader，但任然实现ResourceLoaderAware接口允许重写运行时的ApplicationContext
 * implement ResourceLoaderAware to allow for overriding when running in an
 * ApplicationContext. See ReloadableResourceBundleMessageSource for an example.
 *
 * <p>A passed-in ResourceLoader can also be checked for the
 * <b>ResourcePatternResolver</b> interface and cast accordingly, to be able
 * to resolve resource patterns into arrays of Resource objects. This will always
 * work when running in an ApplicationContext (the context interface extends
 * ResourcePatternResolver). Use a PathMatchingResourcePatternResolver as default.
 * See also the {@code ResourcePatternUtils.getResourcePatternResolver} method.
 *
 * <p>As alternative to a ResourcePatternResolver dependency, consider exposing
 * bean properties of type Resource array, populated via pattern Strings with
 * automatic type conversion by the bean factory.
 *
 * @since 10.03.2004
 * @see ApplicationContextAware
 * @see org.springframework.beans.factory.InitializingBean
 * @see org.springframework.core.io.Resource
 * @see org.springframework.core.io.support.ResourcePatternResolver
 * @see org.springframework.core.io.support.ResourcePatternUtils#getResourcePatternResolver
 * @see org.springframework.core.io.DefaultResourceLoader
 * @see org.springframework.core.io.support.PathMatchingResourcePatternResolver
 * @see org.springframework.context.support.ReloadableResourceBundleMessageSource
 */
public interface ResourceLoaderAware extends Aware {

	/**
	 * Set the ResourceLoader that this object runs in. 设置此对象运行的ResourceLoader
	 * <p>This might be a ResourcePatternResolver, which can be checked
	 * through {@code instanceof ResourcePatternResolver}. See also the
	 * {@code ResourcePatternUtils.getResourcePatternResolver} method. 
	 * <p>Invoked after population of normal bean properties but before an init callback
	 * like InitializingBean's {@code afterPropertiesSet} or a custom init-method.
	 * Invoked before ApplicationContextAware's {@code setApplicationContext}.
	 * @param resourceLoader ResourceLoader object to be used by this object
	 * @see org.springframework.core.io.support.ResourcePatternResolver
	 * @see org.springframework.core.io.support.ResourcePatternUtils#getResourcePatternResolver
	 */
	void setResourceLoader(ResourceLoader resourceLoader);

}


```

## ServletConfigAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified of the
 * {@link ServletConfig} (typically determined断定 by the {@link WebApplicationContext})
 * that it runs in.
 * 只有在特定于servlet中实际运行时才会满足
 * <p>Note: Only satisfied if actually running within a Servlet-specific
 * WebApplicationContext. Otherwise, no ServletConfig will be set.
 *
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 2.0
 * @see ServletContextAware
 */
public interface ServletConfigAware extends Aware {

	/**
	 * Set the {@link ServletConfig} that this object runs in.
	 * <p>Invoked after population of normal bean properties but before an init
	 * callback like InitializingBean's {@code afterPropertiesSet} or a
	 * custom init-method. Invoked after ApplicationContextAware's
	 * {@code setApplicationContext}.
	 * @param servletConfig ServletConfig object to be used by this object
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
	 */
	void setServletConfig(ServletConfig servletConfig);

}
```

## ServletContextAware

```java
/**
 * Interface to be implemented by any object that wishes to be notified of the
 * {@link ServletContext} (typically determined by the {@link WebApplicationContext})
 * that it runs in.
 *
 * @author Juergen Hoeller
 * @author Chris Beams
 * @since 12.03.2004
 * @see ServletConfigAware
 */
public interface ServletContextAware extends Aware {

	/**
	 * Set the {@link ServletContext} that this object runs in. 置此对象运行的 ServletContext
	 * <p>Invoked after population of normal bean properties but before an init
	 * callback like InitializingBean's {@code afterPropertiesSet} or a
	 * custom init-method. Invoked after ApplicationContextAware's
	 * {@code setApplicationContext}.
	 * @param servletContext ServletContext object to be used by this object
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.context.ApplicationContextAware#setApplicationContext
	 */
	void setServletContext(ServletContext servletContext);

}
```