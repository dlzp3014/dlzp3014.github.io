---
layout: post
title:  "Spring源码-ConverterFactory<S, R>转换器工厂"
date:   2019-07-11 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Core
---

* content
{:toc}

ConverterFactory<S, R>用于获取Converter<S, T>接口的实现，主要目的将对象S转换为R的子类，实现`区间`的转换，Spring内部提供的实现有：

- CharacterToNumberFactory 将字符转换为Number的子类

- IntegerToEnumConverterFactory 将整型转换为enum的子类

- NumberToNumberConverterFactory 将Number类型转换为Number的子类

- StringToEnumConverterFactory 将字符串类型转换为enum的子类

- StringToEnumIgnoringCaseConverterFactory 将字符串转换为enum的子类，忽略大小写【SpringBoot提供的实现】

- StringToNumberConverterFactory 将字符串类型转换为Number的子类

可以自定义转换工厂，实现自定义的转换器。具体参考：[SpringBoot中通用Enum枚举值绑定](/2019/07/11/spring-boot-enum-json-view/)实现







## ConverterFactory<S, R>接口定义

```java
/**
 * A factory for "ranged" converters that can convert objects from S to subtypes of R.
 *
 * <p>Implementations may additionally implement {@link ConditionalConverter}.
 *
 * @since 3.0
 * @see ConditionalConverter 条件转换
 * @param <S> the source type converters created by this factory can convert from 源类型
 * @param <R> the target range (or base) type converters created by this factory can convert to; 目标类型或者其子类
 * for example {@link Number} for a set of number subtypes.
 */
public interface ConverterFactory<S, R> {

	/**
	 * Get the converter 获取转换器 to convert from S to target type T, where T is also an instance of R. T也可为R的实例
	 * @param <T> the target type
	 * @param targetType the target type to convert to
	 * @return a converter from S to T
	 */
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);

}

```





## Spring提供的实现


### CharacterToNumberFactory

从一个字符转换为任何标准的Number的实现，支持 Byte, Short, Integer, Float, Double, Long, BigInteger, BigDecimal类型，委托给NumberUtils#convertNumberToTargetClass(Number, Class)执行转换，具体实现查看：[Spring源码-NumberUtils](/2019/07/15/spring-NumberUtils/)


```java
/**
 * Converts from a Character to any JDK-standard Number implementation. Number类型的实现
 *
 * <p>Support Number classes including Byte, Short, Integer, Float, Double, Long, BigInteger, BigDecimal. This class
 * delegates to {@link NumberUtils#convertNumberToTargetClass(Number, Class)} to perform the conversion.
 *
 * @since 3.0
 * @see java.lang.Byte
 * @see java.lang.Short
 * @see java.lang.Integer
 * @see java.lang.Long
 * @see java.math.BigInteger
 * @see java.lang.Float
 * @see java.lang.Double
 * @see java.math.BigDecimal
 * @see NumberUtils
 */
final class CharacterToNumberFactory implements ConverterFactory<Character, Number> {

	@Override
	public <T extends Number> Converter<Character, T> getConverter(Class<T> targetType) { //获取转换器
		return new CharacterToNumber<>(targetType);
	}

	//静态内部类
	private static final class CharacterToNumber<T extends Number> implements Converter<Character, T> {
		//目标类型
		private final Class<T> targetType;

		public CharacterToNumber(Class<T> targetType) {
			this.targetType = targetType;
		}

		@Override
		public T convert(Character source) {
			return NumberUtils.convertNumberToTargetClass((short) source.charValue(), this.targetType);
		}
	}

}


```


### IntegerToEnumConverterFactory

将整型转换为枚举，从0开始

```java
/**
 * Converts from a Integer to a {@link java.lang.Enum} by calling {@link Class#getEnumConstants()}.
 *
 * @since 4.3
 */
@SuppressWarnings({"unchecked", "rawtypes"})
final class IntegerToEnumConverterFactory implements ConverterFactory<Integer, Enum> {

	@Override
	public <T extends Enum> Converter<Integer, T> getConverter(Class<T> targetType) {
		return new IntegerToEnum(ConversionUtils.getEnumType(targetType)); //获取目标枚举类型
	}


	private class IntegerToEnum<T extends Enum> implements Converter<Integer, T> {

		private final Class<T> enumType;

		public IntegerToEnum(Class<T> enumType) {
			this.enumType = enumType;
		}

		@Override
		public T convert(Integer source) {
			return this.enumType.getEnumConstants()[source]; //直接访问数组的index
		}
	}

}

```


### NumberToNumberConverterFactory

将JDK标准的Number实现转换为其他Number的实现类型

```java
/**
 * Converts from any JDK-standard Number implementation to any other JDK-standard Number implementation.
 *
 * <p>Support Number classes including Byte, Short, Integer, Float, Double, Long, BigInteger, BigDecimal. This class
 * delegates to {@link NumberUtils#convertNumberToTargetClass(Number, Class)} to perform the conversion.
 *
 * @author Keith Donald
 * @since 3.0
 * @see java.lang.Byte
 * @see java.lang.Short
 * @see java.lang.Integer
 * @see java.lang.Long
 * @see java.math.BigInteger
 * @see java.lang.Float
 * @see java.lang.Double
 * @see java.math.BigDecimal
 * @see NumberUtils
 */
final class NumberToNumberConverterFactory implements ConverterFactory<Number, Number>, ConditionalConverter {

	@Override
	public <T extends Number> Converter<Number, T> getConverter(Class<T> targetType) {
		return new NumberToNumber<>(targetType);
	}

	@Override
	public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
		return !sourceType.equals(targetType);
	}


	private static final class NumberToNumber<T extends Number> implements Converter<Number, T> {

		private final Class<T> targetType;

		public NumberToNumber(Class<T> targetType) {
			this.targetType = targetType;
		}

		@Override
		public T convert(Number source) {
			return NumberUtils.convertNumberToTargetClass(source, this.targetType);
		}
	}

}

```


### StringToEnumConverterFactory

将字符转换为枚举，通过调用Enum#valueOf(Class, String)方法来实现

```java
/**
 * Converts from a String to a {@link java.lang.Enum} by calling {@link Enum#valueOf(Class, String)}.
 * @since 3.0
 */
@SuppressWarnings({"unchecked", "rawtypes"})
final class StringToEnumConverterFactory implements ConverterFactory<String, Enum> {

	@Override
	public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToEnum(ConversionUtils.getEnumType(targetType));
	}


	private class StringToEnum<T extends Enum> implements Converter<String, T> {

		private final Class<T> enumType;

		public StringToEnum(Class<T> enumType) {
			this.enumType = enumType;
		}

		@Override
		public T convert(String source) {
			if (source.isEmpty()) {
				// It's an empty enum identifier: reset the enum value to null. 源数据不存在时，返回null
				return null;
			}
			return (T) Enum.valueOf(this.enumType, source.trim());
		}
	}

}
```

### StringToNumberConverterFactory

字符串String类型转换为Number的子类

```java
/**
 * Converts from a String any JDK-standard Number implementation.
 *
 * <p>Support Number classes including Byte, Short, Integer, Float, Double, Long, BigInteger, BigDecimal. This class
 * delegates to {@link NumberUtils#parseNumber(String, Class)} to perform the conversion.
 * @since 3.0
 * @see java.lang.Byte
 * @see java.lang.Short
 * @see java.lang.Integer
 * @see java.lang.Long
 * @see java.math.BigInteger
 * @see java.lang.Float
 * @see java.lang.Double
 * @see java.math.BigDecimal
 * @see NumberUtils
 */
final class StringToNumberConverterFactory implements ConverterFactory<String, Number> {

	@Override
	public <T extends Number> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToNumber<>(targetType);
	}


	private static final class StringToNumber<T extends Number> implements Converter<String, T> {

		private final Class<T> targetType;

		public StringToNumber(Class<T> targetType) {
			this.targetType = targetType;
		}

		@Override
		public T convert(String source) {
			if (source.isEmpty()) {
				return null;
			}
			return NumberUtils.parseNumber(source, this.targetType);
		}
	}

}

```
## SpringBoot提供的实现

StringToEnumIgnoringCaseConverterFactory：忽略大小写实现String->enum的转换

```java
/**
 * Converts from a String to a {@link java.lang.Enum} by calling searching matching enum
 * names (ignoring case).
 *
 * @author Phillip Webb
 */
@SuppressWarnings({ "unchecked", "rawtypes" })
final class StringToEnumIgnoringCaseConverterFactory
		implements ConverterFactory<String, Enum> {

	@Override
	public <T extends Enum> Converter<String, T> getConverter(Class<T> targetType) {
		Class<?> enumType = targetType;
		while (enumType != null && !enumType.isEnum()) { //循环查找目标类型父类的枚举值
			enumType = enumType.getSuperclass();
		}
		Assert.notNull(enumType, () -> "The target type " + targetType.getName()
				+ " does not refer to an enum");
		return new StringToEnum(enumType);
	}

	private class StringToEnum<T extends Enum> implements Converter<String, T> {

		private final Class<T> enumType;//目标枚举类型

		StringToEnum(Class<T> enumType) {
			this.enumType = enumType;
		}

		@Override
		public T convert(String source) {
			if (source.isEmpty()) {
				return null;
			}
			source = source.trim();
			try {
				return (T) Enum.valueOf(this.enumType, source); //全字符串匹配转换，出现异常时，处理大小写的问题
			}
			catch (Exception ex) {
				return findEnum(source);
			}
		}

		private T findEnum(String source) {
			String name = getLettersAndDigits(source); //获取字母和数字
			for (T candidate : (Set<T>) EnumSet.allOf(this.enumType)) { //枚举类属性集合
				if (getLettersAndDigits(candidate.name()).equals(name)) {
					return candidate;
				}
			}
			throw new IllegalArgumentException("No enum constant "
					+ this.enumType.getCanonicalName() + "." + source);
		}


		private String getLettersAndDigits(String name) {
			StringBuilder canonicalName = new StringBuilder(name.length());
			name.chars().map((c) -> (char) c).filter(Character::isLetterOrDigit) //过滤是否为字符或者数字
					.map(Character::toLowerCase).forEach(canonicalName::append);//转换为小写合并
			return canonicalName.toString();
		}

	}

}

```



