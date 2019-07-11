---
layout: post
title:  "Spring源码-ClassUtils"
date:   2019-05-08 23:57:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}

ClassUtils主要用于处理Class<?>，提供的静态属性及方法如下：






- 属性

```java
/** Suffix for array class names: "[]" */
	public static final String ARRAY_SUFFIX = "[]";

	/** Prefix for internal array class names: "[" */ 原始数据类型数组
	private static final String INTERNAL_ARRAY_PREFIX = "[";

	/** Prefix for internal non-primitive array class names: "[L" */
	private static final String NON_PRIMITIVE_ARRAY_PREFIX = "[L";

	/** The package separator character: '.' */
	private static final char PACKAGE_SEPARATOR = '.';

	/** The path separator character: '/' */
	private static final char PATH_SEPARATOR = '/';

	/** The inner class separator character: '$' */
	private static final char INNER_CLASS_SEPARATOR = '$';

	/** The CGLIB class separator: "$$" */
	public static final String CGLIB_CLASS_SEPARATOR = "$$";

	/** The ".class" file suffix */
	public static final String CLASS_FILE_SUFFIX = ".class";


	/**
	 * Map with primitive wrapper type as key 原始包装类型 and corresponding primitive 原始类型
	 * type as value, for example: Integer.class -> int.class.
	 */
	private static final Map<Class<?>, Class<?>> primitiveWrapperTypeMap = new IdentityHashMap<>(8);

	/**
	 * Map with primitive type as key and corresponding wrapper
	 * type as value, for example: int.class -> Integer.class. 
	 */
	private static final Map<Class<?>, Class<?>> primitiveTypeToWrapperMap = new IdentityHashMap<>(8);

	/**
	 * Map with primitive type name as key and corresponding primitive 原始类型"名"作为key,相应的原始类型作为值
	 * type as value, for example: "int" -> "int.class".
	 */
	private static final Map<String, Class<?>> primitiveTypeNameMap = new HashMap<>(32);

	/**
	 * Map with common Java language class name as key and corresponding Class as value. Java公共类缓存
	 * Primarily for efficient deserialization of remote invocations.
	 */
	private static final Map<String, Class<?>> commonClassCache = new HashMap<>(64);

	/**
	 * Common Java language interfaces which are supposed to be ignored
	 * when searching for 'primary' user-level interfaces.
	 */
	private static final Set<Class<?>> javaLanguageInterfaces;

```

- 初始化方法

```java
static {
		//填充原始包装类型
		primitiveWrapperTypeMap.put(Boolean.class, boolean.class);
		primitiveWrapperTypeMap.put(Byte.class, byte.class);
		primitiveWrapperTypeMap.put(Character.class, char.class);
		primitiveWrapperTypeMap.put(Double.class, double.class);
		primitiveWrapperTypeMap.put(Float.class, float.class);
		primitiveWrapperTypeMap.put(Integer.class, int.class);
		primitiveWrapperTypeMap.put(Long.class, long.class);
		primitiveWrapperTypeMap.put(Short.class, short.class);

		// Map entry iteration is less expensive to initialize than forEach with lambdas
		for (Map.Entry<Class<?>, Class<?>> entry : primitiveWrapperTypeMap.entrySet()) {
			primitiveTypeToWrapperMap.put(entry.getValue(), entry.getKey());
			registerCommonClasses(entry.getKey());
		}

		Set<Class<?>> primitiveTypes = new HashSet<>(32);
		primitiveTypes.addAll(primitiveWrapperTypeMap.values());
		Collections.addAll(primitiveTypes, boolean[].class, byte[].class, char[].class,
				double[].class, float[].class, int[].class, long[].class, short[].class);
		primitiveTypes.add(void.class);
		for (Class<?> primitiveType : primitiveTypes) {
			primitiveTypeNameMap.put(primitiveType.getName(), primitiveType);
		}

		registerCommonClasses(Boolean[].class, Byte[].class, Character[].class, Double[].class,
				Float[].class, Integer[].class, Long[].class, Short[].class);
		registerCommonClasses(Number.class, Number[].class, String.class, String[].class,
				Class.class, Class[].class, Object.class, Object[].class);
		registerCommonClasses(Throwable.class, Exception.class, RuntimeException.class,
				Error.class, StackTraceElement.class, StackTraceElement[].class);
		registerCommonClasses(Enum.class, Iterable.class, Iterator.class, Enumeration.class,
				Collection.class, List.class, Set.class, Map.class, Map.Entry.class, Optional.class);

		Class<?>[] javaLanguageInterfaceArray = {Serializable.class, Externalizable.class,
				Closeable.class, AutoCloseable.class, Cloneable.class, Comparable.class};
		registerCommonClasses(javaLanguageInterfaceArray);
		javaLanguageInterfaces = new HashSet<>(Arrays.asList(javaLanguageInterfaceArray));
	}


	/**
	 * Register the given common classes with the ClassUtils cache.
	 */
	private static void registerCommonClasses(Class<?>... commonClasses) {
		for (Class<?> clazz : commonClasses) {
			commonClassCache.put(clazz.getName(), clazz);
		}
	}
```




- getDefaultClassLoader()：获取默认的类加载器

```java
	/**
	 * Return the default ClassLoader to use: typically the thread context 代表线程上下文件加载器
	 * ClassLoader, if available; the ClassLoader that loaded the ClassUtils ClassLoader加载ClassUtils类
	 * class will be used as fallback.
	 * <p>Call this method if you intend  打算to use the thread context ClassLoader
	 * in a scenario 方案 where you clearly prefer a non-null ClassLoader reference:
	 * for example, for class path resource loading (but not necessarily for
	 * {@code Class.forName}, which accepts a {@code null} ClassLoader
	 * reference as well).
	 * @return the default ClassLoader (only {@code null} if even the system
	 * ClassLoader isn't accessible)
	 * @see Thread#getContextClassLoader()
	 * @see ClassLoader#getSystemClassLoader()
	 */
	@Nullable
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
			//线程上下文的类加载器
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class. 没有线程上下文类加载器时，使用类当前类的加载器
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader 引导类加载器
				try {
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}
```

- overrideThreadContextClassLoader(@Nullable ClassLoader classLoaderToUse)：使用环境的bean类加载器 重写线程上下文的类加载器

```java
	/**
	 * Override the thread context ClassLoader with the environment's bean ClassLoader 通过环境bean的类加载器重新线程上下文类加载器，如果bean ClassLoader 不等于线程上下文ClassLoader
	 * if necessary, i.e. if the bean ClassLoader is not equivalent to the thread
	 * context ClassLoader already.
	 * @param classLoaderToUse the actual ClassLoader to use for the thread context 用于线程上下文的实际类加载器
	 * @return the original thread context ClassLoader, or {@code null} if not overridden
	 */
	@Nullable
	public static ClassLoader overrideThreadContextClassLoader(@Nullable ClassLoader classLoaderToUse) {
		Thread currentThread = Thread.currentThread();//线程上下
		ClassLoader threadContextClassLoader = currentThread.getContextClassLoader(); //类加载器
		if (classLoaderToUse != null && !classLoaderToUse.equals(threadContextClassLoader)) { //是否相等
			currentThread.setContextClassLoader(classLoaderToUse);
			return threadContextClassLoader;
		}
		else {
			return null;
		}
	}
```

- forName(String name, @Nullable ClassLoader classLoader)：获取Class 实例

```java
	/**
	 * Replacement for {@code Class.forName()} 替换Class.forName()方法 that also returns Class instances 返回类的实例
	 * for primitives (e.g. "int") and array class names (e.g. "String[]"). 对于原始类型或者数组类型
	 * Furthermore 此外, it is also capable of resolving inner class names in Java source 在Java 源风格中，能够解析内部类
	 * style (e.g. "java.lang.Thread.State" instead of "java.lang.Thread$State").
	 * @param name the name of the Class
	 * @param classLoader the class loader to use
	 * (may be {@code null}, which indicates the default class loader)
	 * @return a class instance for the supplied name 提供名称的类实例
	 * @throws ClassNotFoundException if the class was not found
	 * @throws LinkageError if the class file could not be loaded
	 * @see Class#forName(String, boolean, ClassLoader)
	 */
	public static Class<?> forName(String name, @Nullable ClassLoader classLoader)
			throws ClassNotFoundException, LinkageError {

		Assert.notNull(name, "Name must not be null");

		Class<?> clazz = resolvePrimitiveClassName(name); //解析原始类型
		if (clazz == null) {
			clazz = commonClassCache.get(name); //公共类型
		}
		if (clazz != null) {
			return clazz;
		}

		// "java.lang.String[]" style arrays
		if (name.endsWith(ARRAY_SUFFIX)) { //数组
			String elementClassName = name.substring(0, name.length() - ARRAY_SUFFIX.length());//元素类名称
			Class<?> elementClass = forName(elementClassName, classLoader); 
			return Array.newInstance(elementClass, 0).getClass(); //数组实例化
		}

		// "[Ljava.lang.String;" style arrays
		if (name.startsWith(NON_PRIMITIVE_ARRAY_PREFIX) && name.endsWith(";")) {
			String elementName = name.substring(NON_PRIMITIVE_ARRAY_PREFIX.length(), name.length() - 1);
			Class<?> elementClass = forName(elementName, classLoader);
			return Array.newInstance(elementClass, 0).getClass();
		}

		// "[[I" or "[[Ljava.lang.String;" style arrays
		if (name.startsWith(INTERNAL_ARRAY_PREFIX)) {
			String elementName = name.substring(INTERNAL_ARRAY_PREFIX.length());
			Class<?> elementClass = forName(elementName, classLoader);
			return Array.newInstance(elementClass, 0).getClass();
		}

		ClassLoader clToUse = classLoader;
		if (clToUse == null) {
			clToUse = getDefaultClassLoader();
		}
		try {
			return (clToUse != null ? clToUse.loadClass(name) : Class.forName(name));
		}
		catch (ClassNotFoundException ex) {
			int lastDotIndex = name.lastIndexOf(PACKAGE_SEPARATOR);
			if (lastDotIndex != -1) {
				String innerClassName =
						name.substring(0, lastDotIndex) + INNER_CLASS_SEPARATOR + name.substring(lastDotIndex + 1);
				try {
					return (clToUse != null ? clToUse.loadClass(innerClassName) : Class.forName(innerClassName));
				}
				catch (ClassNotFoundException ex2) {
					// Swallow - let original exception get through
				}
			}
			throw ex;
		}
	}
```

- resolveClassName(String className, @Nullable ClassLoader classLoader)：根据class name解析Class 实例

```java
/**
	 * Resolve the given class name into a Class instance. Supports 解析类名为类实例，支持原始类型和数组类名
	 * primitives (like "int") and array class names (like "String[]").
	 * <p>This is effectively  实际上equivalent to the {@code forName}
	 * method with the same arguments, with the only difference being
	 * the exceptions thrown in case of class loading failure. 类加载失败时抛出异常
	 * @param className the name of the Class
	 * @param classLoader the class loader to use
	 * (may be {@code null}, which indicates the default class loader)
	 * @return a class instance for the supplied name
	 * @throws IllegalArgumentException if the class name was not resolvable
	 * (that is, the class could not be found or the class file could not be loaded)
	 * @see #forName(String, ClassLoader)
	 */
	public static Class<?> resolveClassName(String className, @Nullable ClassLoader classLoader)
			throws IllegalArgumentException {

		try {
			return forName(className, classLoader);
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalArgumentException("Could not find class [" + className + "]", ex);
		}
		catch (LinkageError err) {
			throw new IllegalArgumentException("Unresolvable class definition for class [" + className + "]", err);
		}
	}
```

- isPresent(String className, @Nullable ClassLoader classLoader)：通过给定的className确认对应的Class是否存在或者能够加载

```java
	/**
	 * Determine whether the {@link Class} identified by the supplied name is present 通过提供的类名，确认class是否存在，且能够加载
	 * and can be loaded. Will return {@code false} if either the class or
	 * one of its dependencies is not present or cannot be loaded.
	 * @param className the name of the class to check
	 * @param classLoader the class loader to use
	 * (may be {@code null} which indicates the default class loader)
	 * @return whether the specified class is present
	 */
	public static boolean isPresent(String className, @Nullable ClassLoader classLoader) {
		try {
			forName(className, classLoader);
			return true;
		}
		catch (Throwable ex) {
			// Class or one of its dependencies is not present...
			return false;
		}
	}
```

- isVisible(Class<?> clazz, @Nullable ClassLoader classLoader)：检查给定的Class是否在给定的ClassLoader可见

```java
/**
	 * Check whether the given class is visible in the given ClassLoader.
	 * @param clazz the class to check (typically an interface)
	 * @param classLoader the ClassLoader to check against
	 * (may be {@code null} in which case this method will always return {@code true})
	 */
	public static boolean isVisible(Class<?> clazz, @Nullable ClassLoader classLoader) {
		if (classLoader == null) { 
			return true;
		}
		try {
			if (clazz.getClassLoader() == classLoader) { //类的加载器相同
				return true;
			}
		}
		catch (SecurityException ex) {
			// Fall through to loadable check below
		}

		// Visible if same Class can be loaded from given ClassLoader
		return isLoadable(clazz, classLoader);
	}

```

- isCacheSafe(Class<?> clazz, @Nullable ClassLoader classLoader)：检查给定的类在给定上下文中是否安全缓存，由它由给定的类加载器或它的父类加载器加载决定

```java
	/**
	 * Check whether the given class is cache-safe in the given context,
	 * i.e. whether it is loaded by the given ClassLoader or a parent of it.通过给定的类加载器是否可加载或者使用父类加载器
	 * @param clazz the class to analyze
	 * @param classLoader the ClassLoader to potentially cache metadata in
	 * (may be {@code null} which indicates the system class loader)
	 */
	public static boolean isCacheSafe(Class<?> clazz, @Nullable ClassLoader classLoader) {
		Assert.notNull(clazz, "Class must not be null");
		try {
			ClassLoader target = clazz.getClassLoader();
			// Common cases
			if (target == classLoader || target == null) {
				return true;
			}
			if (classLoader == null) {
				return false;
			}
			// Check for match in ancestors -> positive 祖先：向上查找
			ClassLoader current = classLoader;
			while (current != null) {
				current = current.getParent();
				if (current == target) {
					return true;
				}
			}
			// Check for match in children -> negative //向下不匹配查找
			while (target != null) {
				target = target.getParent();
				if (target == classLoader) {
					return false;
				}
			}
		}
		catch (SecurityException ex) {
			// Fall through to loadable check below
		}

		// Fallback for ClassLoaders without parent/child relationship: 关系
		// safe if same Class can be loaded from given ClassLoader
		return (classLoader != null && isLoadable(clazz, classLoader));
	}
```


- isLoadable(Class<?> clazz, ClassLoader classLoader)：检查给定的类在给定的类加载器中是否可加载

```java
	/**
	 * Check whether the given class is loadable in the given ClassLoader.
	 * @param clazz the class to check (typically an interface)
	 * @param classLoader the ClassLoader to check against
	 * @since 5.0.6
	 */
	private static boolean isLoadable(Class<?> clazz, ClassLoader classLoader) {
		try {
			return (clazz == classLoader.loadClass(clazz.getName()));
			// Else: different class with same name found
		}
		catch (ClassNotFoundException ex) {
			// No corresponding class found at all
			return false;
		}
	}
```


- Class<?> resolvePrimitiveClassName(@Nullable String name)：将给定的类名解析为基本数据类型

```java
	/**
	 * Resolve the given class name as primitive class, if appropriate, 解析给定的类名作为原始类，
	 * according to the JVM's naming rules for primitive classes. 根据JVM的命名规则
	 * <p>Also supports the JVM's internal class names for primitive arrays. 支持原始类型数组
	 * Does <i>not</i> support the "[]" suffix notation 符号 for primitive arrays;
	 * this is only supported by {@link #forName(String, ClassLoader)}.
	 * @param name the name of the potentially primitive class
	 * @return the primitive class, or {@code null} if the name does not denote 表示
	 * a primitive class or primitive array class
	 */
	@Nullable
	public static Class<?> resolvePrimitiveClassName(@Nullable String name) {
		Class<?> result = null;
		// Most class names will be quite long, considering that they
		// SHOULD sit 位于 in a package, so a length check is worthwhile.
		if (name != null && name.length() <= 8) {
			// Could be a primitive - likely.
			result = primitiveTypeNameMap.get(name);
		}
		return result;
	}

```

- isPrimitiveWrapper(Class<?> clazz)：检查给定的类是否表示原始包装类型 【 Boolean, Byte, Character, Short, Integer, Long, Float, or Double】

```java
	/**
	 * Check if the given class represents a primitive wrapper,
	 * i.e. Boolean, Byte, Character, Short, Integer, Long, Float, or Double.
	 * @param clazz the class to check
	 * @return whether the given class is a primitive wrapper class
	 */
	public static boolean isPrimitiveWrapper(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return primitiveWrapperTypeMap.containsKey(clazz);
	}
```

- isPrimitiveOrWrapper(Class<?> clazz)：检查给定的类是否表示基元原始类型

```java
	/**
	 * Check if the given class represents a primitive (i.e. boolean, byte,
	 * char, short, int, long, float, or double) or a primitive wrapper
	 * (i.e. Boolean, Byte, Character, Short, Integer, Long, Float, or Double).
	 * @param clazz the class to check
	 * @return whether the given class is a primitive or primitive wrapper class
	 */
	public static boolean isPrimitiveOrWrapper(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return (clazz.isPrimitive() || isPrimitiveWrapper(clazz));
	}
```

- isPrimitiveArray(Class<?> clazz)：是否为原始数组类型

```java
	/**
	 * Check if the given class represents an array of primitives,
	 * i.e. boolean, byte, char, short, int, long, float, or double.
	 * @param clazz the class to check
	 * @return whether the given class is a primitive array class
	 */
	public static boolean isPrimitiveArray(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return (clazz.isArray() && clazz.getComponentType().isPrimitive());
	}

```


- isPrimitiveWrapperArray(Class<?> clazz)：是否为原始包装数组类型

```java
	/**
	 * Check if the given class represents an array of primitive wrappers,
	 * i.e. Boolean, Byte, Character, Short, Integer, Long, Float, or Double.
	 * @param clazz the class to check
	 * @return whether the given class is a primitive wrapper array class
	 */
	public static boolean isPrimitiveWrapperArray(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return (clazz.isArray() && isPrimitiveWrapper(clazz.getComponentType()));
	}
```

- resolvePrimitiveIfNecessary(Class<?> clazz)：从给定的Class解析出原始数据类型

```java
	/**
	 * Resolve the given class if it is a primitive class, 如果是原始类型，解析
	 * returning the corresponding primitive wrapper type instead. 返回对于的原始包装类型代替
	 * @param clazz the class to check
	 * @return the original class, or a primitive wrapper for the original primitive type
	 */
	public static Class<?> resolvePrimitiveIfNecessary(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return (clazz.isPrimitive() && clazz != void.class ? primitiveTypeToWrapperMap.get(clazz) : clazz);
	}
```

- isAssignable(Class<?> lhsType, Class<?> rhsType)：检查是否可以将右侧类型分配(赋值)给左侧类型，主要用于java中的多态

```java
	/**
	 * Check if the right-hand side type may be assigned to 指向 the left-hand side
	 * type, assuming setting by reflection. Considers primitive wrapper 考虑原始包装类型作为指向对于的原始类型
	 * classes as assignable to the corresponding primitive types.
	 * @param lhsType the target type
	 * @param rhsType the value type that should be assigned to the target type
	 * @return if the target type is assignable from the value type
	 * @see TypeUtils#isAssignable
	 */
	public static boolean isAssignable(Class<?> lhsType, Class<?> rhsType) {
		Assert.notNull(lhsType, "Left-hand side type must not be null");
		Assert.notNull(rhsType, "Right-hand side type must not be null");
		if (lhsType.isAssignableFrom(rhsType)) {
			return true;
		}
		if (lhsType.isPrimitive()) {
			Class<?> resolvedPrimitive = primitiveWrapperTypeMap.get(rhsType);
			if (lhsType == resolvedPrimitive) {
				return true;
			}
		}
		else {
			Class<?> resolvedWrapper = primitiveTypeToWrapperMap.get(rhsType);
			if (resolvedWrapper != null && lhsType.isAssignableFrom(resolvedWrapper)) {
				return true;
			}
		}
		return false;
	}
```

- isAssignableValue(Class<?> type, @Nullable Object value):确定给定类型是否可以从给定值分配，即将value赋值给type

```java
	/**
	 * Determine if the given type is assignable from the given value,
	 * assuming setting by reflection. Considers primitive wrapper classes
	 * as assignable to the corresponding primitive types.
	 * @param type the target type
	 * @param value the value that should be assigned to the type
	 * @return if the type is assignable from the value
	 */
	public static boolean isAssignableValue(Class<?> type, @Nullable Object value) {
		Assert.notNull(type, "Type must not be null");
		return (value != null ? isAssignable(type, value.getClass()) : !type.isPrimitive());
	}
```

- convertResourcePathToClassName(String resourcePath):将的资源路径转"/"转换为基于全限定的类名的"."

```java
/**
	 * Convert a "."-based fully qualified class name 全限类名 to a "/"-based resource path. 资源路径
	 * @param className the fully qualified class name
	 * @return the corresponding resource path, pointing to the class
	 */
	public static String convertClassNameToResourcePath(String className) {
		Assert.notNull(className, "Class name must not be null");
		return className.replace(PACKAGE_SEPARATOR, PATH_SEPARATOR);
	}
```

- convertClassNameToResourcePath(String className):与convertResourcePathToClassName相反，将全限类名转换为路径

```java
	/**
	 * Convert a "/"-based resource path to a "."-based fully qualified class name.
	 * @param resourcePath the resource path pointing to a class
	 * @return the corresponding fully qualified class name
	 */
	public static String convertResourcePathToClassName(String resourcePath) {
		Assert.notNull(resourcePath, "Resource path must not be null");
		return resourcePath.replace(PATH_SEPARATOR, PACKAGE_SEPARATOR);
	}
```

- addResourcePathToPackagePath(Class<?> clazz, String resourceName)：添加资源路径到类对应的包名路径

```java
	/**
	 * Return a path suitable for use with {@code ClassLoader.getResource} 返回适当的路径，用于ClassLoader.getResource
	 * (also suitable for use with {@code Class.getResource} by prepending a
	 * slash ('/') to the return value). Built by taking the package of the specified 
	 * class file, converting all dots ('.') to slashes ('/'), adding a trailing slash
	 * if necessary, and concatenating 连接 the specified resource name to this.
	 * <br/>As such, this function may be used to build a path suitable for 这个方法可能被用于构建适当的路径，用于加载资源文件
	 * loading a resource file that is in the same package as a class file,相同的包作为类文件，
	 * although {@link org.springframework.core.io.ClassPathResource} is usually
	 * even more convenient.
	 * @param clazz the Class whose package will be used as the base
	 * @param resourceName the resource name to append. A leading slash is optional.
	 * @return the built-up resource path
	 * @see ClassLoader#getResource
	 * @see Class#getResource
	 */
	public static String addResourcePathToPackagePath(Class<?> clazz, String resourceName) {
		Assert.notNull(resourceName, "Resource name must not be null");
		if (!resourceName.startsWith("/")) {
			return classPackageAsResourcePath(clazz) + '/' + resourceName;
		}
		return classPackageAsResourcePath(clazz) + resourceName;
	}
```


- classPackageAsResourcePath(@Nullable Class<?> clazz)：返回类的包名作为资源路径

```java
	/**
	 * Given an input class object, return a string which consists of the 有类的包名和路径名组成
	 * class's package name as a pathname, i.e., all dots ('.') are replaced by 
	 * slashes ('/'). Neither a leading nor trailing slash is added. The result
	 * could be concatenated with a slash and the name of a resource and fed
	 * directly to {@code ClassLoader.getResource()}. For it to be fed to
	 * {@code Class.getResource} instead, a leading slash would also have
	 * to be prepended to the returned value.
	 * @param clazz the input class. A {@code null} value or the default
	 * (empty) package will result in an empty string ("") being returned.
	 * @return a path which represents the package name
	 * @see ClassLoader#getResource
	 * @see Class#getResource
	 */
	public static String classPackageAsResourcePath(@Nullable Class<?> clazz) {
		if (clazz == null) {
			return "";
		}
		String className = clazz.getName();
		int packageEndIndex = className.lastIndexOf(PACKAGE_SEPARATOR); //不包含包名
		if (packageEndIndex == -1) {
			return "";
		}
		String packageName = className.substring(0, packageEndIndex); //截取，替换
		return packageName.replace(PACKAGE_SEPARATOR, PATH_SEPARATOR);
	}
```

- classNamesToString(Class<?>... classes)：将类名转换为字符串，以","分割

	classNamesToString(@Nullable Collection<Class<?>> classes)

```java
/**
	 * Build a String that consists of the names of the classes/interfaces 构建一个String有类或者接口名组成
	 * in the given array.
	 * <p>Basically like {@code AbstractCollection.toString()}, but stripping
	 * the "class "/"interface " prefix before every class name.
	 * @param classes an array of Class objects
	 * @return a String of form "[com.foo.Bar, com.foo.Baz]"
	 * @see java.util.AbstractCollection#toString()
	 */
	public static String classNamesToString(Class<?>... classes) {
		return classNamesToString(Arrays.asList(classes));
	}

	/**
	 * Build a String that consists of the names of the classes/interfaces
	 * in the given collection.
	 * <p>Basically like {@code AbstractCollection.toString()}, but stripping
	 * the "class "/"interface " prefix before every class name.
	 * @param classes a Collection of Class objects (may be {@code null})
	 * @return a String of form "[com.foo.Bar, com.foo.Baz]"
	 * @see java.util.AbstractCollection#toString()
	 */
	public static String classNamesToString(@Nullable Collection<Class<?>> classes) {
		if (CollectionUtils.isEmpty(classes)) {
			return "[]";
		}
		StringBuilder sb = new StringBuilder("[");
		for (Iterator<Class<?>> it = classes.iterator(); it.hasNext(); ) {
			Class<?> clazz = it.next();
			sb.append(clazz.getName());
			if (it.hasNext()) {
				sb.append(", ");
			}
		}
		sb.append("]");
		return sb.toString();
	}
```

- toClassArray(Collection<Class<?>> collection)：将类集合转换为类数组

```java

	/**
	 * Copy the given {@code Collection} into a {@code Class} array. 
	 * <p>The {@code Collection} must contain {@code Class} elements only. 集合只能包含Class元素
	 * @param collection the {@code Collection} to copy
	 * @return the {@code Class} array
	 * @since 3.1
	 * @see StringUtils#toStringArray
	 */
	public static Class<?>[] toClassArray(Collection<Class<?>> collection) {
		return collection.toArray(new Class<?>[0]);
	}
```
- getAllInterfaces(Object instance)：获取实例对象的所有接口

	getAllInterfacesForClass(Class<?> clazz):获取类的所有接口
	getAllInterfacesForClass(Class<?> clazz, @Nullable ClassLoader classLoader)：
	getAllInterfacesAsSet(Object instance):
	getAllInterfacesForClassAsSet(Class<?> clazz)

```java
	/**
	 * Return all interfaces that the given instance implements as an array,
	 * including ones implemented by superclasses.
	 * @param instance the instance to analyze for interfaces
	 * @return all interfaces that the given instance implements as an array
	 */
	public static Class<?>[] getAllInterfaces(Object instance) {
		Assert.notNull(instance, "Instance must not be null");
		return getAllInterfacesForClass(instance.getClass());
	}

	/**
	 * Return all interfaces that the given class implements as an array,
	 * including ones implemented by superclasses.
	 * <p>If the class itself is an interface, it gets returned as sole interface.
	 * @param clazz the class to analyze for interfaces
	 * @return all interfaces that the given object implements as an array
	 */
	public static Class<?>[] getAllInterfacesForClass(Class<?> clazz) {
		return getAllInterfacesForClass(clazz, null);
	}

	/**
	 * Return all interfaces that the given class implements as an array,
	 * including ones implemented by superclasses.
	 * <p>If the class itself is an interface, it gets returned as sole interface.
	 * @param clazz the class to analyze for interfaces
	 * @param classLoader the ClassLoader that the interfaces need to be visible in
	 * (may be {@code null} when accepting all declared interfaces)
	 * @return all interfaces that the given object implements as an array
	 */
	public static Class<?>[] getAllInterfacesForClass(Class<?> clazz, @Nullable ClassLoader classLoader) {
		return toClassArray(getAllInterfacesForClassAsSet(clazz, classLoader));
	}

	/**
	 * Return all interfaces that the given instance implements as a Set,
	 * including ones implemented by superclasses.
	 * @param instance the instance to analyze for interfaces
	 * @return all interfaces that the given instance implements as a Set
	 */
	public static Set<Class<?>> getAllInterfacesAsSet(Object instance) {
		Assert.notNull(instance, "Instance must not be null");
		return getAllInterfacesForClassAsSet(instance.getClass());
	}

	/**
	 * Return all interfaces that the given class implements as a Set,
	 * including ones implemented by superclasses.
	 * <p>If the class itself is an interface, it gets returned as sole interface.
	 * @param clazz the class to analyze for interfaces
	 * @return all interfaces that the given object implements as a Set
	 */
	public static Set<Class<?>> getAllInterfacesForClassAsSet(Class<?> clazz) {
		return getAllInterfacesForClassAsSet(clazz, null);
	}


```

- getAllInterfacesForClassAsSet(Class<?> clazz, @Nullable ClassLoader classLoader)

```java
	/**
	 * Return all interfaces that the given class implements as a Set, 返回给定类的所有接口作为一个set集合
	 * including ones implemented by superclasses.
	 * <p>If the class itself is an interface, it gets returned as sole interface.
	 * @param clazz the class to analyze for interfaces
	 * @param classLoader the ClassLoader that the interfaces need to be visible in 可访问
	 * (may be {@code null} when accepting all declared interfaces)
	 * @return all interfaces that the given object implements as a Set
	 */
	public static Set<Class<?>> getAllInterfacesForClassAsSet(Class<?> clazz, @Nullable ClassLoader classLoader) {
		Assert.notNull(clazz, "Class must not be null");
		//接口且可使用classloader加载
		if (clazz.isInterface() && isVisible(clazz, classLoader)) {
			return Collections.singleton(clazz);
		}
		Set<Class<?>> interfaces = new LinkedHashSet<>();
		Class<?> current = clazz;
		while (current != null) {
			Class<?>[] ifcs = current.getInterfaces(); //当前类接口可访问
			for (Class<?> ifc : ifcs) {
				if (isVisible(ifc, classLoader)) {
					interfaces.add(ifc);
				}
			}
			current = current.getSuperclass();
		}
		return interfaces;
	}
```

- createCompositeInterface(Class<?>[] interfaces, @Nullable ClassLoader classLoader)：为给定的接口创建一个合成的接口类。即创建接口的代理对象

```java
	/**
	 * Create a composite interface Class for the given interfaces,
	 * implementing the given interfaces in one single Class. 类中的给定接口
	 * <p>This implementation builds a JDK proxy class for the given interfaces. 构建给定接口的JDK代理类
	 * @param interfaces the interfaces to merge
	 * @param classLoader the ClassLoader to create the composite Class in
	 * @return the merged interface as Class
	 * @throws IllegalArgumentException if the specified interfaces expose
	 * conflicting method signatures (or a similar constraint is violated)
	 * @see java.lang.reflect.Proxy#getProxyClass
	 */
	@SuppressWarnings("deprecation")
	public static Class<?> createCompositeInterface(Class<?>[] interfaces, @Nullable ClassLoader classLoader) {
		Assert.notEmpty(interfaces, "Interfaces must not be empty");
		return Proxy.getProxyClass(classLoader, interfaces);
	}
```


- determineCommonAncestor(@Nullable Class<?> clazz1, @Nullable Class<?> clazz2)：确定给定类是否有公共祖先，即获取两个指定类共同的父类

```java
	/**
	 * Determine the common ancestor of the given classes, if any. 确定给定类的公共祖先
	 * @param clazz1 the class to introspect
	 * @param clazz2 the other class to introspect
	 * @return the common ancestor (i.e. common superclass, one interface
	 * extending the other), or {@code null} if none found. If any of the
	 * given classes is {@code null}, the other class will be returned.
	 * @since 3.2.6
	 */
	@Nullable
	public static Class<?> determineCommonAncestor(@Nullable Class<?> clazz1, @Nullable Class<?> clazz2) {
		if (clazz1 == null) {
			return clazz2;
		}
		if (clazz2 == null) {
			return clazz1;
		}
		if (clazz1.isAssignableFrom(clazz2)) {
			return clazz1;
		}
		if (clazz2.isAssignableFrom(clazz1)) {
			return clazz2;
		}
		Class<?> ancestor = clazz1; //循环层级查找
		do {
			ancestor = ancestor.getSuperclass();
			if (ancestor == null || Object.class == ancestor) {
				return null;
			}
		}
		while (!ancestor.isAssignableFrom(clazz2));
		return ancestor;
	}
```

- public static boolean isJavaLanguageInterface(Class<?> ifc)：java提供的公共接口

```java
	/**
	 * Determine whether the given interface is a common Java language interface:
	    ：确定给定接口是否是公共Java语言接口
	 * {@link Serializable}, {@link Externalizable}, {@link Closeable}, {@link AutoCloseable},
	 * {@link Cloneable}, {@link Comparable} - all of which can be ignored when looking
	 * for 'primary' user-level interfaces. Common characteristics: no service-level
	 * operations, no bean property methods, no default methods.
	 * @param ifc the interface to check
	 * @since 5.0.3
	 */
	public static boolean isJavaLanguageInterface(Class<?> ifc) {
		return javaLanguageInterfaces.contains(ifc);
	}
```

- isInnerClass(Class<?> clazz)：非静态内部类

```java
	/**
	 * Determine if the supplied class is an <em>inner class</em>,
	 * i.e. a non-static member of an enclosing class. 封闭类的非静态成员
	 * @return {@code true} if the supplied class is an inner class
	 * @since 5.0.5
	 * @see Class#isMemberClass()
	 */
	public static boolean isInnerClass(Class<?> clazz) {
		return (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers()));
	}
```

- isCglibProxy(Object object)：Cglib代理

```java
	/**
	 * Check whether the given object is a CGLIB proxy. 检查给定对象是否是CGLIB代理
	 * @param object the object to check
	 * @see #isCglibProxyClass(Class)
	 * @see org.springframework.aop.support.AopUtils#isCglibProxy(Object)
	 */
	public static boolean isCglibProxy(Object object) {
		return isCglibProxyClass(object.getClass());
	}
```

- isCglibProxyClass(@Nullable Class<?> clazz)

```java
	/**
	 * Check whether the specified class is a CGLIB-generated class. CGLIB生成类
	 * @param clazz the class to check
	 * @see #isCglibProxyClassName(String)
	 */
	public static boolean isCglibProxyClass(@Nullable Class<?> clazz) {
		return (clazz != null && isCglibProxyClassName(clazz.getName()));
	}
```

- isCglibProxyClassName(@Nullable String className)

```java
	/**
	 * Check whether the specified class name is a CGLIB-generated class.
	 * @param className the class name to check
	 */
	public static boolean isCglibProxyClassName(@Nullable String className) {
		return (className != null && className.contains(CGLIB_CLASS_SEPARATOR));
	}
```

- getUserClass(Object instance)：返回给定实例的用户定义类

```java
	/**
	 * Return the user-defined class for the given instance: usually simply
	 * the class of the given instance, but the original class in case of a
	 * CGLIB-generated subclass.
	 * @param instance the instance to check
	 * @return the user-defined class
	 */
	public static Class<?> getUserClass(Object instance) {
		Assert.notNull(instance, "Instance must not be null");
		return getUserClass(instance.getClass());
	}
```

- getUserClass(Class<?> clazz)

```java
	/**
	 * Return the user-defined class for the given class: usually simply the given
	 * class, but the original class in case of a CGLIB-generated subclass.
	 * @param clazz the class to check
	 * @return the user-defined class
	 */
	public static Class<?> getUserClass(Class<?> clazz) {
		if (clazz.getName().contains(CGLIB_CLASS_SEPARATOR)) {
			Class<?> superclass = clazz.getSuperclass();  Cglib代理类返回父类
			if (superclass != null && superclass != Object.class) {
				return superclass;
			}
		}
		return clazz;
	}

```

- getDescriptiveType(@Nullable Object value)：类描述

```java
	/**
	 * Return a descriptive name for the given object's type: usually simply 返回给定对象类型的一个描述名，通常是简单的类名
	 * the class name, but component type class name + "[]" for arrays,
	 * and an appended list of implemented interfaces for JDK proxies.
	 * @param value the value to introspect
	 * @return the qualified name of the class
	 */
	@Nullable
	public static String getDescriptiveType(@Nullable Object value) {
		if (value == null) {
			return null;
		}
		Class<?> clazz = value.getClass(); //对象的类
		if (Proxy.isProxyClass(clazz)) { //是否为jdk代理类
			StringBuilder result = new StringBuilder(clazz.getName());
			result.append(" implementing ");
			Class<?>[] ifcs = clazz.getInterfaces();
			for (int i = 0; i < ifcs.length; i++) {
				result.append(ifcs[i].getName());
				if (i < ifcs.length - 1) {
					result.append(',');
				}
			}
			return result.toString();
		}
		else {
			return clazz.getTypeName();
		}
	}
```

- matchesTypeName(Class<?> clazz, @Nullable String typeName)：检查给定的类是否匹配用户指定的类型名称

```java
	/**
	 * Check whether the given class matches the user-specified type name. 检查给定类是否匹配用于指定的类型名
	 * @param clazz the class to check
	 * @param typeName the type name to match
	 */
	public static boolean matchesTypeName(Class<?> clazz, @Nullable String typeName) {
		return (typeName != null &&
				//类型名或者类的简单名
				(typeName.equals(clazz.getTypeName()) || typeName.equals(clazz.getSimpleName())));
	}
```

- getShortName(String className)：获取类的短名字，不包含包名

```java

	/**
	 * Get the class name without the qualified package name.
	 * @param className the className to get the short name for
	 * @return the class name of the class without the package name
	 * @throws IllegalArgumentException if the className is empty
	 */
	public static String getShortName(String className) {
		Assert.hasLength(className, "Class name must not be empty");
		int lastDotIndex = className.lastIndexOf(PACKAGE_SEPARATOR); "."
		int nameEndIndex = className.indexOf(CGLIB_CLASS_SEPARATOR); "$$" Cglib代理类名
		if (nameEndIndex == -1) {
			nameEndIndex = className.length();
		}
		String shortName = className.substring(lastDotIndex + 1, nameEndIndex);
		shortName = shortName.replace(INNER_CLASS_SEPARATOR, PACKAGE_SEPARATOR);"'$'"
		return shortName;
	}
```

- getShortName(Class<?> clazz) 

```java
	/**
	 * Get the class name without the qualified package name. 获取类名不包括全限包名
	 * @param clazz the class to get the short name for
	 * @return the class name of the class without the package name
	 */
	public static String getShortName(Class<?> clazz) {
		return getShortName(getQualifiedName(clazz));
	}
```

- getShortNameAsProperty(Class<?> clazz)：内部类时不包含外部类名

```java
	/**
	 * Return the short string name of a Java class in uncapitalized JavaBeans 返回java类的短字符串名，
	 * property format. Strips the outer class name in case of an inner class. 在内部类的情况下去掉外部类名
	 * @param clazz the class
	 * @return the short name rendered in a standard JavaBeans property format 标准JavaBeans 属性格式
	 * @see java.beans.Introspector#decapitalize(String)
	 */
	public static String getShortNameAsProperty(Class<?> clazz) {
		String shortName = getShortName(clazz);
		int dotIndex = shortName.lastIndexOf(PACKAGE_SEPARATOR);
		shortName = (dotIndex != -1 ? shortName.substring(dotIndex + 1) : shortName);
		return Introspector.decapitalize(shortName);//首字母大写
	}
```

- getClassFileName(Class<?> clazz)：获取类的文件

```java
	/**
	 * Determine the name of the class file, relative to the containing 确认类文件的名称，相对于所含包
	 * package: e.g. "String.class"
	 * @param clazz the class
	 * @return the file name of the ".class" file
	 */
	public static String getClassFileName(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		String className = clazz.getName();
		int lastDotIndex = className.lastIndexOf(PACKAGE_SEPARATOR); //截取最后"."
		return className.substring(lastDotIndex + 1) + CLASS_FILE_SUFFIX; 
	}
```

- getPackageName(Class<?> clazz)：获取包名

```java
	/**
	 * Determine the name of the package of the given class,
	 * e.g. "java.lang" for the {@code java.lang.String} class.
	 * @param clazz the class
	 * @return the package name, or the empty String if the class
	 * is defined in the default package
	 */
	public static String getPackageName(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return getPackageName(clazz.getName());
	}
```

- getPackageName(String fqClassName)

```java
	/**
	 * Determine the name of the package of the given fully-qualified class name,给定的完全限定类名
	 * e.g. "java.lang" for the {@code java.lang.String} class name.
	 * @param fqClassName the fully-qualified class name
	 * @return the package name, or the empty String if the class
	 * is defined in the default package
	 */
	public static String getPackageName(String fqClassName) {
		Assert.notNull(fqClassName, "Class name must not be null");
		int lastDotIndex = fqClassName.lastIndexOf(PACKAGE_SEPARATOR); "."是否包含
		return (lastDotIndex != -1 ? fqClassName.substring(0, lastDotIndex) : "");
	}
```

- getQualifiedName(Class<?> clazz)：返回类的限定名

```java
	/**
	 * Return the qualified name of the given class: usually simply 通常是简单的类名，
	 * the class name, but component type class name + "[]" for arrays. 组合类名为[] 
	 * @param clazz the class
	 * @return the qualified name of the class
	 */
	public static String getQualifiedName(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		return clazz.getTypeName();
	}
```

- getQualifiedMethodName(Method method)：返回方法的限定名

```java
	/**
	 * Return the qualified name of the given method, consisting of
	 * fully qualified interface/class name + "." + method name.
	 * @param method the method
	 * @return the qualified name of the method
	 */
	public static String getQualifiedMethodName(Method method) {
		return getQualifiedMethodName(method, null);
	}
```

- getQualifiedMethodName(Method method, @Nullable Class<?> clazz)

```java
	/**
	 * Return the qualified name of the given method, consisting of 返回给定方法的限定名
	 * fully qualified interface/class name + "." + method name. 组成全限定名
	 * @param method the method
	 * @param clazz the clazz that the method is being invoked on
	 * (may be {@code null} to indicate the method's declaring class) 指示方法的声明类
	 * @return the qualified name of the method
	 * @since 4.3.4
	 */
	public static String getQualifiedMethodName(Method method, @Nullable Class<?> clazz) {
		Assert.notNull(method, "Method must not be null");
		//clazz==null，从Method中获取方法所在的声明类
		return (clazz != null ? clazz : method.getDeclaringClass()).getName() + '.' + method.getName();
	}
``

- hasConstructor(Class<?> clazz, Class<?>... paramTypes)：确定给定类是否具有给定签名的公共构造函数

```java
	/**
	 * Determine whether the given class has a public constructor with the given signature.
	 * <p>Essentially translates {@code NoSuchMethodException} to "false".
	 * @param clazz the clazz to analyze
	 * @param paramTypes the parameter types of the method
	 * @return whether the class has a corresponding constructor
	 * @see Class#getMethod
	 */
	public static boolean hasConstructor(Class<?> clazz, Class<?>... paramTypes) {
		return (getConstructorIfAvailable(clazz, paramTypes) != null);
	}
```

- Constructor<T> getConstructorIfAvailable(Class<T> clazz, Class<?>... paramTypes)：获取构造方法

```java
	/**
	 * Determine whether the given class has a public constructor with the given signature, 确定给定类是否具有具有给定签名的公共构造函数
	 * and return it if available (else return {@code null}).
	 * <p>Essentially translates {@code NoSuchMethodException} to {@code null}.
	 * @param clazz the clazz to analyze
	 * @param paramTypes the parameter types of the method
	 * @return the constructor, or {@code null} if not found
	 * @see Class#getConstructor
	 */
	@Nullable
	public static <T> Constructor<T> getConstructorIfAvailable(Class<T> clazz, Class<?>... paramTypes) {
		Assert.notNull(clazz, "Class must not be null");
		try {
			return clazz.getConstructor(paramTypes);
		}
		catch (NoSuchMethodException ex) {
			return null;
		}
	}
```

- hasMethod(Class<?> clazz, String methodName, Class<?>... paramTypes)：是否有指定参数类型的方法

```java
	/**
	 * Determine whether the given class has a public method with the given signature.
	 * <p>Essentially translates {@code NoSuchMethodException} to "false".
	 * @param clazz the clazz to analyze
	 * @param methodName the name of the method
	 * @param paramTypes the parameter types of the method
	 * @return whether the class has a corresponding method
	 * @see Class#getMethod
	 */
	public static boolean hasMethod(Class<?> clazz, String methodName, Class<?>... paramTypes) {
		return (getMethodIfAvailable(clazz, methodName, paramTypes) != null); //调用是否有可获取指定的方法
	}
```

- getMethod(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes)：获取方法指定参数类型的方法，与getMethodIfAvailable最大的区别在于当前方法未找到时抛出IllegalStateException参数非法异常

```java
/**
	 * Determine whether the given class has a public method with the given signature,
	 * and return it if available (else throws an {@code IllegalStateException}).
	 * <p>In case of any signature specified, only returns the method if there is a
	 * unique candidate, i.e. a single public method with the specified name.
	 * <p>Essentially translates {@code NoSuchMethodException} to {@code IllegalStateException}.
	 * @param clazz the clazz to analyze
	 * @param methodName the name of the method
	 * @param paramTypes the parameter types of the method
	 * (may be {@code null} to indicate any signature)
	 * @return the method (never {@code null})
	 * @throws IllegalStateException if the method has not been found
	 * @see Class#getMethod
	 */
	public static Method getMethod(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes) {
		Assert.notNull(clazz, "Class must not be null");
		Assert.notNull(methodName, "Method name must not be null");
		if (paramTypes != null) {
			try {
				return clazz.getMethod(methodName, paramTypes);
			}
			catch (NoSuchMethodException ex) {
				throw new IllegalStateException("Expected method not found: " + ex);
			}
		}
		else {
			Set<Method> candidates = new HashSet<>(1);
			Method[] methods = clazz.getMethods();
			for (Method method : methods) {
				if (methodName.equals(method.getName())) {
					candidates.add(method);
				}
			}
			if (candidates.size() == 1) {
				return candidates.iterator().next();
			}
			else if (candidates.isEmpty()) {
				throw new IllegalStateException("Expected method not found: " + clazz.getName() + '.' + methodName);
			}
			else {
				throw new IllegalStateException("No unique method found: " + clazz.getName() + '.' + methodName);
			}
		}
	}
```

- getMethodIfAvailable(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes):根据方法名及方法中的参数，从Class中获取public Method对象

```java
	/**
	 * Determine whether the given class has a public method 公共方法通过给定的签名 with the given signature,
	 * and return it if available (else return {@code null}).
	 * <p>In case of 假设 any signature specified, only returns the method if there is a 唯一候选
	 * unique candidate, i.e. a single public method with the specified name.
	 * <p>Essentially 本来 translates {@code NoSuchMethodException} to {@code null}. 转换NoSuchMethodException异常为null
	 * @param clazz the clazz to analyze
	 * @param methodName the name of the method
	 * @param paramTypes the parameter types of the method
	 * (may be {@code null} to indicate any signature)
	 * @return the method, or {@code null} if not found
	 * @see Class#getMethod
	 */
	@Nullable
	public static Method getMethodIfAvailable(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes) {
		Assert.notNull(clazz, "Class must not be null");
		Assert.notNull(methodName, "Method name must not be null");
		if (paramTypes != null) { //参数列表不为null时，直接从clazz.getMethod获取，没有时返回null
			try {
				return clazz.getMethod(methodName, paramTypes);
			}
			catch (NoSuchMethodException ex) {
				return null;
			}
		}
		else {
			Set<Method> candidates = new HashSet<>(1);//候选方法集合
			Method[] methods = clazz.getMethods();
			for (Method method : methods) {
				if (methodName.equals(method.getName())) {
					candidates.add(method);
				}
			}
			if (candidates.size() == 1) {
				return candidates.iterator().next();
			}
			//多个候选方法时返回null
			return null;
		}
	}
```

- getMethodCountForName(Class<?> clazz, String methodName)：获取方法名对应的个数。分别从clazz.getDeclaredMethods()类的声明方法、clazz.getInterfaces()类的接口、clazz.getSuperclass()类的父类中累加计算

```java
	/**
	 * Return the number of methods with a given name (with any argument types),
	 * for the given class and/or its superclasses. Includes non-public methods.
	 * @param clazz	the clazz to check
	 * @param methodName the name of the method
	 * @return the number of methods with the given name
	 */
	public static int getMethodCountForName(Class<?> clazz, String methodName) {
		Assert.notNull(clazz, "Class must not be null");
		Assert.notNull(methodName, "Method name must not be null");
		int count = 0;
		Method[] declaredMethods = clazz.getDeclaredMethods();
		for (Method method : declaredMethods) {
			if (methodName.equals(method.getName())) {
				count++;
			}
		}
		Class<?>[] ifcs = clazz.getInterfaces();
		for (Class<?> ifc : ifcs) {
			count += getMethodCountForName(ifc, methodName);
		}
		if (clazz.getSuperclass() != null) {
			count += getMethodCountForName(clazz.getSuperclass(), methodName);
		}
		return count;
	}

```

- hasAtLeastOneMethodWithName(Class<?> clazz, String methodName)：至少包含一个指定的方法名,递归调用

```java
	/**
	 * Does the given class or one of its superclasses at least have one or more 给定的类或者父类之一，只是包含一个或者更多指定的方法名
	 * methods with the supplied name (with any argument types)?
	 * Includes non-public methods. 包含非公开的方法
	 * @param clazz	the clazz to check
	 * @param methodName the name of the method
	 * @return whether there is at least one method with the given name
	 */
	public static boolean hasAtLeastOneMethodWithName(Class<?> clazz, String methodName) {
		Assert.notNull(clazz, "Class must not be null");
		Assert.notNull(methodName, "Method name must not be null");
		Method[] declaredMethods = clazz.getDeclaredMethods(); //获取类中所有的方法描述，遍历是否与methodName相同
		for (Method method : declaredMethods) {
			if (method.getName().equals(methodName)) {
				return true;
			}
		}
		Class<?>[] ifcs = clazz.getInterfaces(); //获取类中的接口，递归调用
		for (Class<?> ifc : ifcs) {
			if (hasAtLeastOneMethodWithName(ifc, methodName)) {
				return true;
			}
		}
		//父类
		return (clazz.getSuperclass() != null && hasAtLeastOneMethodWithName(clazz.getSuperclass(), methodName));
	}

```

- getMostSpecificMethod(Method method, @Nullable Class<?> targetClass)：获取指定的方法，主要使用重目标类的clazz.getMethod或者Java reflective

```java
	/**
	 * Given a method, which may come from an interface, and a target class used 给定一个可能来自接口，目标类用于当前反射调用的方法
	 * in the current reflective invocation, find the corresponding target method 找出相关的目标方法
	 * if there is one. E.g. the method may be {@code IFoo.bar()} and the
	 * target class may be {@code DefaultFoo}. In this case, the method may be
	 * {@code DefaultFoo.bar()}. This enables attributes on that method to be found. 这样就可以找到该方法上的属性
	 		与……形成对照
	 * <p><b>NOTE:</b> In contrast to {@link org.springframework.aop.support.AopUtils#getMostSpecificMethod},
	 * this method does <i>not</i> resolve Java 5 bridge methods automatically. 不自动解析Java 5 提供的自动的桥接方法
	 * Call {@link org.springframework.core.BridgeMethodResolver#findBridgedMethod}
	 * if bridge method resolution is desirable 可取 (e.g. for obtaining metadata from 从原始方法定义中获取原始元数据
	 * the original method definition).
	 * <p><b>NOTE:</b> Since Spring 3.1.1, if Java security settings disallow 不准许 reflective
	 * access (e.g. calls to {@code Class#getDeclaredMethods} etc, this implementation
	 * will fall back to returning the originally provided method. 最初的方法方法提供者
	 * @param method the method to be invoked, which may come from an interface 要调用的方法，可能来自于接口
	 * @param targetClass the target class for the current invocation  用于当前调用的目标类
	 * (may be {@code null} or may not even implement the method)
	 * @return the specific target method, or the original method if the 指定的目标方法，或者最初的方法
	 * {@code targetClass} does not implement it
	 */
	public static Method getMostSpecificMethod(Method method, @Nullable Class<?> targetClass) {
		//targetClass 不为null 且，方法声明的类不是目标类，且方法可重写
		if (targetClass != null && targetClass != method.getDeclaringClass() && isOverridable(method, targetClass)) {
			try {
				if (Modifier.isPublic(method.getModifiers())) { //public 修饰符的方法
					try {
						//从目标类中根据方法名，及方法参数获取方法目标类对应的方法
						return targetClass.getMethod(method.getName(), method.getParameterTypes());
					}
					catch (NoSuchMethodException ex) {
						return method;
					}
				}
				else {
					//反射目标类查找类中的方法
					Method specificMethod =
							ReflectionUtils.findMethod(targetClass, method.getName(), method.getParameterTypes());
					return (specificMethod != null ? specificMethod : method);
				}
			}
			catch (SecurityException ex) {
				// Security settings are disallowing reflective access; fall back to 'method' below.
			}
		}
		return method;
	}

```
- isUserLevelMethod(Method method)：是否为用户声明的方法

```java
/**
 * Determine whether the given method is declared by the user or at least pointing to 用户声明或者至少指向用户声明的方法
 * a user-declared method.
 * <p>Checks {@link Method#isSynthetic()} 合成的 (for implementation methods) as well as the
 * {@code GroovyObject} interface (for interface methods; on an implementation class,
 * implementations of the {@code GroovyObject} methods will be marked as synthetic anyway).
 * Note that, despite being synthetic, bridge methods ({@link Method#isBridge()}) are considered
 它们最终指向一个用户声明的泛型方法
 * as user-level methods since they are eventually 最终 pointing to a user-declared generic method. 
 * @param method the method to check
 * @return {@code true} if the method can be considered as user-declared; [@code false} otherwise
 */
public static boolean isUserLevelMethod(Method method) {
	Assert.notNull(method, "Method must not be null");
	return (method.isBridge() || (!method.isSynthetic() && !isGroovyObjectMethod(method)));
}

```

- isGroovyObjectMethod(Method method)

```java
private static boolean isGroovyObjectMethod(Method method) {
	return method.getDeclaringClass().getName().equals("groovy.lang.GroovyObject");
}
```

- isOverridable(Method method, @Nullable Class<?> targetClass)：确定给定的方法在给定的目标类中是否可重写

```java
/**
 * Determine whether the given method is overridable in the given target class. 确定给定的方法是否在给定义的类可重新
 * @param method the method to check
 * @param targetClass the target class to check against
 */
private static boolean isOverridable(Method method, @Nullable Class<?> targetClass) {
	if (Modifier.isPrivate(method.getModifiers())) { //方法修饰符为私有返回false
		return false;
	}
	if (Modifier.isPublic(method.getModifiers()) || Modifier.isProtected(method.getModifiers())) { //方法修饰符为public或者为Protected返回true
		return true;
	}
	return (targetClass == null ||
			//从方法中获取类的描述符再次获取报名与类的报名是否相同
			getPackageName(method.getDeclaringClass()).equals(getPackageName(targetClass)));
}
```

- getStaticMethod(Class<?> clazz, String methodName, Class<?>... args)：获取类中的静态方法

```java

/**
 * Return a public static method of a class. 返回类的public static方法
 * @param clazz the class which defines the method 定义方法的类
 * @param methodName the static method name 静态方法名
 * @param args the parameter types to the method 方法的参数类型
 * @return the static method, or {@code null} if no static method was found
 * @throws IllegalArgumentException if the method name is blank or the clazz is null 如果方法名为空或clazz为空
 */
@Nullable
public static Method getStaticMethod(Class<?> clazz, String methodName, Class<?>... args) {
	Assert.notNull(clazz, "Class must not be null");
	Assert.notNull(methodName, "Method name must not be null");
	try {
		Method method = clazz.getMethod(methodName, args); //根据方法名和参数从类中获取方法
		return Modifier.isStatic(method.getModifiers()) ? method : null; //获取方法的修饰符
	}
	catch (NoSuchMethodException ex) {
		return null;
	}
}

```

