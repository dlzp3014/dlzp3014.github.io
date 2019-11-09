---
layout: post
title:  "Spring源码-InitializingBean(Bean)初始化接口"
date:   2019-10-27 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-MVC
---

* content
{:toc}


InitializingBean接口用于处理Bean的自定义初始化操作(对Bean的属性执行二次处理)或者只是为了检查Bean是否设置了所有必需的属性。使用的场景为：当BeanFactory将bean创建成功后，并设置完Bean的所有属性后，这个时候又需要做出自定义的动作(反应)，如原先设置的属性进行加密或者替换、检查等操作时，这时就可以让的Bean实现InitializingBean接口

InitializingBean接口的另一种可代替方案为指定一个自定义的init方法，然后在XML中使用`init-method`属性进行执行或者使用Java EE中提供的@PostConstruct注解




- 使用示例：对已初始化的属性进行加密


```java
public class PersonBean implements InitializingBean {


    private String password;

    public PersonBean(String password) {
        this.password = password;
    }


    @Override
    public void afterPropertiesSet() throws Exception {
        this.password = Base64.getEncoder().encodeToString(password.getBytes("UTF-8"));
    }

    public String getPassword() {
        return this.password;
    }
}
```

```java
@ComponentScan()
@Configuration
public class BeanMain {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(BeanMain.class);
        PersonBean bean = applicationContext.getBean(PersonBean.class);
        Assert.assertEquals(bean.getPassword(), Base64.getEncoder().encodeToString("123456".getBytes()));
    }

    @Bean
    public PersonBean personBean() {
        return new PersonBean("123456");
    }

}
```


- InitializingBean接口定义如下：

```java
/**
 * Interface to be implemented by beans that need to react once all their properties
 * have been set by a {@link BeanFactory}: e.g. to perform custom initialization,
 * or merely to check that all mandatory properties have been set.
 *
 * <p>An alternative to implementing {@code InitializingBean} is specifying a custom
 * init method, for example in an XML bean definition. For a list of all bean
 * lifecycle methods, see the {@link BeanFactory BeanFactory javadocs}.
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see DisposableBean
 * @see org.springframework.beans.factory.config.BeanDefinition#getPropertyValues()
 * @see org.springframework.beans.factory.support.AbstractBeanDefinition#getInitMethodName()
 */
public interface InitializingBean {

	/**
	 * Invoked by the containing {@code BeanFactory} after it has set all bean properties
	 * and satisfied {@link BeanFactoryAware}, {@code ApplicationContextAware} etc.
	 * <p>This method allows the bean instance to perform validation 校验 of its overall
	 * configuration配置 and final initialization 最终的初始化当Bean的所有属性都已经设置when all bean properties have been set.
	 * @throws Exception in the event of misconfiguration (such as failure to set an
	 * essential property) or if initialization fails for any other reason
	 */
	void afterPropertiesSet() throws Exception;

}
```

- BeanFactory中执行过程：

1): AbstractAutowireCapableBeanFactory#doCreateBean

2):	AbstractAutowireCapableBeanFactory#initializeBean

3): AbstractAutowireCapableBeanFactory#invokeInitMethods

```java
boolean isInitializingBean = (bean instanceof InitializingBean);
if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
	if (logger.isDebugEnabled()) {
		logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
	}
	if (System.getSecurityManager() != null) {
		try {
			AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
				((InitializingBean) bean).afterPropertiesSet();
				return null;
			}, getAccessControlContext());
		}
		catch (PrivilegedActionException pae) {
			throw pae.getException();
		}
	}
	else {
		((InitializingBean) bean).afterPropertiesSet();
	}
}
```