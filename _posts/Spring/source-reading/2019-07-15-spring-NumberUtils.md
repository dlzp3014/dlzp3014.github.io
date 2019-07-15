---
layout: post
title:  "Spring源码-NumberUtils"
date:   2019-07-15 08:38:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}

NumberUtils用于数字转换和解析，主要用于框架内部，包括将Number转换为Number的其他类型和将String转换为Number类型







- 静态属性

```java
/**
 * Miscellaneous utility methods for number conversion and parsing.
 * <p>Mainly for internal use within the framework; consider Apache's
 * Commons Lang for a more comprehensive suite 更全面的套件 of number utilities. 
 *
 * @since 1.1.2
 */
public abstract class NumberUtils {

	private static final BigInteger LONG_MIN = BigInteger.valueOf(Long.MIN_VALUE);

	private static final BigInteger LONG_MAX = BigInteger.valueOf(Long.MAX_VALUE);

	/**
	 * Standard number types (all immutable): 标准number类型
	 * Byte, Short, Integer, Long, BigInteger, Float, Double, BigDecimal.
	 */
	public static final Set<Class<?>> STANDARD_NUMBER_TYPES;

	static {
		Set<Class<?>> numberTypes = new HashSet<>(8);
		numberTypes.add(Byte.class);
		numberTypes.add(Short.class);
		numberTypes.add(Integer.class);
		numberTypes.add(Long.class);
		numberTypes.add(BigInteger.class);
		numberTypes.add(Float.class);
		numberTypes.add(Double.class);
		numberTypes.add(BigDecimal.class);
		STANDARD_NUMBER_TYPES = Collections.unmodifiableSet(numberTypes); //不可变集合
	}

```


- T convertNumberToTargetClass(Number number, Class<T> targetClass)：将数字转换为Number子类型的实例



```java
/**
 * Convert the given number into an instance of the given target class. 
 * @param number the number to convert
 * @param targetClass the target class to convert to
 * @return the converted number
 * @throws IllegalArgumentException if the target class is not supported
 * (i.e. not a standard Number subclass as included in the JDK)
 * @see java.lang.Byte
 * @see java.lang.Short
 * @see java.lang.Integer
 * @see java.lang.Long
 * @see java.math.BigInteger
 * @see java.lang.Float
 * @see java.lang.Double
 * @see java.math.BigDecimal
 */
@SuppressWarnings("unchecked")
public static <T extends Number> T convertNumberToTargetClass(Number number, Class<T> targetClass)
		throws IllegalArgumentException {

	Assert.notNull(number, "Number must not be null");
	Assert.notNull(targetClass, "Target class must not be null");

	if (targetClass.isInstance(number)) { //当前实例强转
		return (T) number;
	}
	else if (Byte.class == targetClass) { //字节
		long value = checkedLongValue(number, targetClass); 
		if (value < Byte.MIN_VALUE || value > Byte.MAX_VALUE) {
			raiseOverflowException(number, targetClass);
		}
		return (T) Byte.valueOf(number.byteValue());
	}
	else if (Short.class == targetClass) { 
		long value = checkedLongValue(number, targetClass);
		if (value < Short.MIN_VALUE || value > Short.MAX_VALUE) {
			raiseOverflowException(number, targetClass);
		}
		return (T) Short.valueOf(number.shortValue());
	}
	else if (Integer.class == targetClass) {
		long value = checkedLongValue(number, targetClass);
		if (value < Integer.MIN_VALUE || value > Integer.MAX_VALUE) {
			raiseOverflowException(number, targetClass);
		}
		return (T) Integer.valueOf(number.intValue());
	}
	else if (Long.class == targetClass) {
		long value = checkedLongValue(number, targetClass);
		return (T) Long.valueOf(value);
	}
	else if (BigInteger.class == targetClass) {
		if (number instanceof BigDecimal) {
			// do not lose precision - use BigDecimal's own conversion 不要失去精度
			return (T) ((BigDecimal) number).toBigInteger();
		}
		else {
			// original value is not a Big* number - use standard long conversion
			return (T) BigInteger.valueOf(number.longValue());
		}
	}
	else if (Float.class == targetClass) {
		return (T) Float.valueOf(number.floatValue());
	}
	else if (Double.class == targetClass) {
		return (T) Double.valueOf(number.doubleValue());
	}
	else if (BigDecimal.class == targetClass) {
		// always use BigDecimal(String) here to avoid unpredictability  不可预测性 of BigDecimal(double)
		// (see BigDecimal javadoc for details)
		return (T) new BigDecimal(number.toString());
	}
	else {
		throw new IllegalArgumentException("Could not convert number [" + number + "] of type [" +
				number.getClass().getName() + "] to unsupported target class [" + targetClass.getName() + "]");
	}
}


/**
 * Check for a {@code BigInteger}/{@code BigDecimal} long overflow 检查BigInteger/BigDecimal 长度溢出
 * before returning the given number as a long value. 在将给定的数字作为长值返回之前
 * @param number the number to convert
 * @param targetClass the target class to convert to
 * @return the long value, if convertible without overflow
 * @throws IllegalArgumentException if there is an overflow
 * @see #raiseOverflowException
 */
private static long checkedLongValue(Number number, Class<? extends Number> targetClass) {
	BigInteger bigInt = null;
	if (number instanceof BigInteger) {
		bigInt = (BigInteger) number;
	}
	else if (number instanceof BigDecimal) {
		bigInt = ((BigDecimal) number).toBigInteger();
	}
	// Effectively analogous to JDK 8's BigInteger.longValueExact()
	if (bigInt != null && (bigInt.compareTo(LONG_MIN) < 0 || bigInt.compareTo(LONG_MAX) > 0)) {
		raiseOverflowException(number, targetClass);
	}
	return number.longValue();
}

/**
 * Raise an <em>overflow</em> exception for the given number and target class.
 * @param number the number we tried to convert
 * @param targetClass the target class we tried to convert to
 * @throws IllegalArgumentException if there is an overflow
 */
private static void raiseOverflowException(Number number, Class<?> targetClass) {
	throw new IllegalArgumentException("Could not convert number [" + number + "] of type [" +
			number.getClass().getName() + "] to target class [" + targetClass.getName() + "]: overflow");
}
```


- T parseNumber(String text, Class<T> targetClass):解析String为Number子类的实例，包括十六进制

```java
/**
 * Parse the given {@code text} into a {@link Number} instance of the given
 * target class, using the corresponding 使用对应的 {@code decode} / {@code valueOf} method.
 * <p>Trims the input {@code String} before attempting to parse the number. 尝试解析时只需trim()操作
 * <p>Supports numbers in hex format (with leading "0x", "0X", or "#") as well. 支持16进制number格式
 * @param text the text to convert
 * @param targetClass the target class to parse into
 * @return the parsed number
 * @throws IllegalArgumentException if the target class is not supported
 * (i.e. not a standard Number subclass as included in the JDK)
 * @see Byte#decode
 * @see Short#decode
 * @see Integer#decode
 * @see Long#decode
 * @see #decodeBigInteger(String)
 * @see Float#valueOf
 * @see Double#valueOf
 * @see java.math.BigDecimal#BigDecimal(String)
 */
@SuppressWarnings("unchecked")
public static <T extends Number> T parseNumber(String text, Class<T> targetClass) {
	Assert.notNull(text, "Text must not be null");
	Assert.notNull(targetClass, "Target class must not be null");
	String trimmed = StringUtils.trimAllWhitespace(text); //处理掉所有空格

	if (Byte.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? Byte.decode(trimmed) : Byte.valueOf(trimmed));
	}
	else if (Short.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? Short.decode(trimmed) : Short.valueOf(trimmed));
	}
	else if (Integer.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? Integer.decode(trimmed) : Integer.valueOf(trimmed));
	}
	else if (Long.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? Long.decode(trimmed) : Long.valueOf(trimmed));
	}
	else if (BigInteger.class == targetClass) {
		return (T) (isHexNumber(trimmed) ? decodeBigInteger(trimmed) : new BigInteger(trimmed));
	}
	else if (Float.class == targetClass) {
		return (T) Float.valueOf(trimmed);
	}
	else if (Double.class == targetClass) {
		return (T) Double.valueOf(trimmed);
	}
	else if (BigDecimal.class == targetClass || Number.class == targetClass) {
		return (T) new BigDecimal(trimmed);
	}
	else {
		throw new IllegalArgumentException(
				"Cannot convert String [" + text + "] to target class [" + targetClass.getName() + "]");
	}
}


/**
 * Determine whether the given {@code value} String indicates a hex number, 确认给定的字符串是否表示一个16进制符
 * i.e. needs to be passed into {@code Integer.decode} instead of  使用decode代替valueOf方法
 * {@code Integer.valueOf}, etc.
 */
private static boolean isHexNumber(String value) {
	int index = (value.startsWith("-") ? 1 : 0);//是否为负数
	//除符号位外，是否已0x/0X/#开始
	return (value.startsWith("0x", index) || value.startsWith("0X", index) || value.startsWith("#", index));
}

/**
 * Decode a {@link java.math.BigInteger} from the supplied {@link String} value.
 * <p>Supports decimal, hex, and octal notation. 支持十、十六、八进制进制
 * @see BigInteger#BigInteger(String, int)
 */
private static BigInteger decodeBigInteger(String value) {
	int radix = 10;
	int index = 0;
	boolean negative = false;//负数

	// Handle minus sign, if present.
	if (value.startsWith("-")) {
		negative = true;
		index++;
	}

	// Handle radix specifier, if present.处理基数是否存在
	if (value.startsWith("0x", index) || value.startsWith("0X", index)) {
		index += 2;
		radix = 16;
	}
	else if (value.startsWith("#", index)) {
		index++;
		radix = 16;
	}
	else if (value.startsWith("0", index) && value.length() > 1 + index) {
		index++;
		radix = 8;
	}

	BigInteger result = new BigInteger(value.substring(index), radix);
	return (negative ? result.negate() : result);
}

```