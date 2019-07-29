---
layout: post
title:  "Spring源码-PropertySources多属性源接口"
date:   2019-07-04 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

PropertySources接口包含了多个PropertySource的实例，主要用于遍历PropertySource，详细说明参考[Spring源码-PropertySource<T\>属性源抽象类](/2019/07/04/spring-PropertySource/)，默认实现为MutablePropertySources多属性源。



## PropertySources:多属性源集合接口定义

PropertySources接口继承Iterable<PropertySource<?>>迭代器，迭代元素为PropertySource<?>,?代表具体的源属性对象


```java
/**
 * Holder containing one or more {@link PropertySource} objects.
 *
 * @author Chris Beams
 * @since 3.1
 * @see PropertySource
 */
public interface PropertySources extends Iterable<PropertySource<?>> {

	/**
	 * Return whether a property source with the given name is contained.
	 * @param name the {@linkplain PropertySource#getName() name of the property source} to find 属性源的名称
	 */
	boolean contains(String name);

	/**
	 * Return the property source with the given name, {@code null} if not found. 根据名字返回属性源
	 * @param name the {@linkplain PropertySource#getName() name of the property source} to find
	 */
	@Nullable
	PropertySource<?> get(String name);

}


```

## MutablePropertySources：多属于源

MutablePropertySource为PropertySources接口的默认实现，允许对包含的属性源进行操作和复制已经存在的PropertySources实例

```java
/**
 * The default implementation of the {@link PropertySources} interface.
 * Allows manipulation of contained property sources and provides a constructor
 * for copying an existing {@code PropertySources} instance.
 *
 * <p>Where <em>precedence</em> is mentioned in methods such as {@link #addFirst}
 * and {@link #addLast}, this is with regard 考虑 to the order顺序 in which property sources
 * will be searched when resolving a given property with a {@link PropertyResolver}.
 * 当使用PropertyResolver解析一个给定的属性时，涉及优先级的问题(通过addFirst、addLast方法改变顺序)
 * @author Chris Beams
 * @author Juergen Hoeller
 * @since 3.1
 * @see PropertySourcesPropertyResolver
 */
public class MutablePropertySources implements PropertySources {

	private final Log logger;

	//属性源列表：写时复制
	private final List<PropertySource<?>> propertySourceList = new CopyOnWriteArrayList<>();


	/**
	 * Create a new {@link MutablePropertySources} object.
	 */
	public MutablePropertySources() {
		this.logger = LogFactory.getLog(getClass());
	}

	/**
	 * Create a new {@code MutablePropertySources} from the given propertySources
	 * object, preserving 保留原始的顺序 the original order of contained {@code PropertySource} objects.
	 */
	public MutablePropertySources(PropertySources propertySources) {
		this();
		for (PropertySource<?> propertySource : propertySources) { //遍历向后添加
			addLast(propertySource);
		}
	}

	/**
	 * Create a new {@link MutablePropertySources} object and inherit the given logger,
	 * usually from an enclosing {@link Environment}.
	 */
	MutablePropertySources(Log logger) {
		this.logger = logger;
	}


	@Override
	public Iterator<PropertySource<?>> iterator() {
		return this.propertySourceList.iterator();
	}

	@Override
	public boolean contains(String name) {
		return this.propertySourceList.contains(PropertySource.named(name));
	}

	@Override
	@Nullable
	public PropertySource<?> get(String name) { 获取集合中的索引
		int index = this.propertySourceList.indexOf(PropertySource.named(name));//PropertySource.named(name)创建ComparisonPropertySource，用于比较
		return (index != -1 ? this.propertySourceList.get(index) : null);
	}


	/**
	 * Add the given property source object with highest precedence. 
	 */
	public void addFirst(PropertySource<?> propertySource) { 添加具有最高优先级的给定属性源对象
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() + "' with highest search precedence");
		}
		removeIfPresent(propertySource); 移除给定的属性源
		this.propertySourceList.add(0, propertySource); 在列表的头添加属性源
	}

	/**
	 * Add the given property source object with lowest precedence.
	 */
	public void addLast(PropertySource<?> propertySource) { 添加具有最低优先级的给定属性源对象
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() + "' with lowest search precedence");
		}
		removeIfPresent(propertySource);
		this.propertySourceList.add(propertySource);在列表的尾部添加属性源
	}

	/**
	 * Add the given property source object with precedence immediately higher
	 * than the named relative property source. 
	 */
	public void addBefore(String relativePropertySourceName, PropertySource<?> propertySource) { 添加优先级立高于指定的相对属性源的给定属性源对象
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() +
					"' with search precedence immediately higher than '" + relativePropertySourceName + "'");
		}
		assertLegalRelativeAddition(relativePropertySourceName, propertySource);//确保给定的属性源是否是自己
		removeIfPresent(propertySource);
		int index = assertPresentAndGetIndex(relativePropertySourceName);//获取相对属性源的索引
		addAtIndex(index, propertySource);//在指定索引上添加
	}

	/**
	 * Add the given property source object with precedence immediately lower
	 * than the named relative property source.
	 */
	public void addAfter(String relativePropertySourceName, PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Adding PropertySource '" + propertySource.getName() +
					"' with search precedence immediately lower than '" + relativePropertySourceName + "'");
		}
		assertLegalRelativeAddition(relativePropertySourceName, propertySource);//确保属性名是否是自己
		removeIfPresent(propertySource);
		int index = assertPresentAndGetIndex(relativePropertySourceName); //获取相对属性名的位置
		addAtIndex(index + 1, propertySource); //在相对位置+1中添加
	}

	/**
	 * Return the precedence of the given property source, {@code -1} if not found. 返回给定属性的优先级
	 */
	public int precedenceOf(PropertySource<?> propertySource) {
		return this.propertySourceList.indexOf(propertySource);
	}

	/**
	 * Remove and return the property source with the given name, {@code null} if not found. 
	 * @param name the name of the property source to find and remove
	 */
	@Nullable
	public PropertySource<?> remove(String name) { 移除给定的属性源名
		if (logger.isDebugEnabled()) {
			logger.debug("Removing PropertySource '" + name + "'");
		}
		int index = this.propertySourceList.indexOf(PropertySource.named(name));
		return (index != -1 ? this.propertySourceList.remove(index) : null);
	}

	/**
	 * Replace the property source with the given name with the given property source object. 替换属性源的名
	 * @param name the name of the property source to find and replace
	 * @param propertySource the replacement property source
	 * @throws IllegalArgumentException if no property source with the given name is present 给定属性源的名不存在时抛出异常
	 * @see #contains
	 */
	public void replace(String name, PropertySource<?> propertySource) {
		if (logger.isDebugEnabled()) {
			logger.debug("Replacing PropertySource '" + name + "' with '" + propertySource.getName() + "'");
		}
		int index = assertPresentAndGetIndex(name);
		this.propertySourceList.set(index, propertySource);
	}

	/**
	 * Return the number of {@link PropertySource} objects contained.
	 */
	public int size() {
		return this.propertySourceList.size();
	}

	@Override
	public String toString() {
		return this.propertySourceList.toString();
	}

	/**
	 * Ensure that the given property source is not being added relative to itself. 确保给定的属性源已经被添加过
	 */
	protected void assertLegalRelativeAddition(String relativePropertySourceName, PropertySource<?> propertySource) {
		String newPropertySourceName = propertySource.getName();
		if (relativePropertySourceName.equals(newPropertySourceName)) {
			throw new IllegalArgumentException(
					"PropertySource named '" + newPropertySourceName + "' cannot be added relative to itself");
		}
	}

	/**
	 * Remove the given property source if it is present. 移除给定的属性源
	 */
	protected void removeIfPresent(PropertySource<?> propertySource) {
		this.propertySourceList.remove(propertySource);
	}

	/**
	 * Add the given property source at a particular index in the list. 在列表中的特定索引处添加给定的属性源
	 */
	private void addAtIndex(int index, PropertySource<?> propertySource) {
		removeIfPresent(propertySource);
		this.propertySourceList.add(index, propertySource);
	}

	/**
	 * Assert that the named property source is present and return its index. 断言属性源名是否存在，否则抛出异常
	 * @param name {@linkplain PropertySource#getName() name of the property source} to find
	 * @throws IllegalArgumentException if the named property source is not present
	 */
	private int assertPresentAndGetIndex(String name) {
		int index = this.propertySourceList.indexOf(PropertySource.named(name));
		if (index == -1) {
			throw new IllegalArgumentException("PropertySource named '" + name + "' does not exist");
		}
		return index;
	}

}


```

