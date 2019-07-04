---
layout: post
title:  "Spring源码-IdGenerator Id生成器接口"
date:   2019-05-27 22:03:00
categories: Spring 
tags: Spring-Source-Reading Spring-Utils
---

* content
{:toc}

IdGenerator接口主要用于生成唯一的Id，Spring提供的默认实现有JdkIdGenerator、SimpleIdGenerator、AlternativeJdkIdGenerator，具体实现过程如下：





## IdGenerator

```java
/**
 * Contract for generating universally unique identifiers {@link UUID (UUIDs)}. 生成意味的标识
 *
 * @since 4.0
 */
public interface IdGenerator {

	/**
	 * Generate a new identifier. 生成一个新的标识符
	 * @return the generated identifier
	 */
	UUID generateId();

}
```

## JdkIdGenerator

JdkIdGenerator内部使用JDK提供的UUID.randomUUID()方法生成UUID

```java
/** 
 * An {@link IdGenerator} that calls {@link java.util.UUID#randomUUID()}.
 *
 * @since 4.1.5
 */
public class JdkIdGenerator implements IdGenerator {

	@Override
	public UUID generateId() {
		return UUID.randomUUID();
	}

}


```

## SimpleIdGenerator

```java
/**
 * A simple {@link IdGenerator} that starts at 1 and increments by 1 with each call. 从1开始，每次调用递增1
 *
 * @since 4.1.5
 */
public class SimpleIdGenerator implements IdGenerator {

	private final AtomicLong mostSigBits = new AtomicLong(0); //Long数据类型的原子操作

	private final AtomicLong leastSigBits = new AtomicLong(0);


	@Override
	public UUID generateId() {
		long leastSigBits = this.leastSigBits.incrementAndGet(); //增加
		if (leastSigBits == 0) {
			this.mostSigBits.incrementAndGet();
		}
		return new UUID(this.mostSigBits.get(), leastSigBits);
	}

}

```

## AlternativeJdkIdGenerator

```java
/**
 * An {@link IdGenerator} that uses {@link SecureRandom} for the initial seed and 使用初始化的种子和
 * {@link Random} thereafter, instead of calling {@link UUID#randomUUID()} every
 * time as {@link org.springframework.util.JdkIdGenerator JdkIdGenerator} does.
 * This provides a better balance between securely random ids and performance.这在安全随机id和性能之间提供了更好的平衡
 *
 * @since 4.0
 */
public class AlternativeJdkIdGenerator implements IdGenerator {

	private final Random random;


	public AlternativeJdkIdGenerator() {
		SecureRandom secureRandom = new SecureRandom(); //安全随机数
		byte[] seed = new byte[8];
		secureRandom.nextBytes(seed);
		this.random = new Random(new BigInteger(seed).longValue());
	}


	@Override
	public UUID generateId() {
		byte[] randomBytes = new byte[16];
		this.random.nextBytes(randomBytes);

		long mostSigBits = 0;
		for (int i = 0; i < 8; i++) {
			mostSigBits = (mostSigBits << 8) | (randomBytes[i] & 0xff);
		}

		long leastSigBits = 0;
		for (int i = 8; i < 16; i++) {
			leastSigBits = (leastSigBits << 8) | (randomBytes[i] & 0xff);
		}

		return new UUID(mostSigBits, leastSigBits);
	}

}

```