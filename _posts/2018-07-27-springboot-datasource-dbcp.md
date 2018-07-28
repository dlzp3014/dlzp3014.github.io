---
layout: post
title:  "SpringBoot配置dbcp2为数据源"
date:   2018-07-27 00:19:00
categories: SpringBoot
tags: SpringBoot DataSource
---

* content
{:toc}



## 定义dbcp2数据源连接属性

application.yml:

```java
spring:
  datasource:
    # 数据源配置
    url: jdbc:mysql://localhost:3306/springboot-demo?characterEncoding=utf8
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: 123456
    #连接池配置
    type: org.apache.commons.dbcp2.BasicDataSource
    dbcp2:
      max-wait-millis: 10000
      max-idle: 50
      min-idle: 5
      max-active: 250
      initial-size: 50
      validation-query: SELECT 1
      connection-properties: characterEncoding=utf8

```

dbcp2详细配置文件请参考[dbcp2数据源配置](/2018/07/27/springboot-datasource-dbcp)

## POM 文件定义

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.dlzp.java</groupId>
		<artifactId>dlzp-spring-boot-parent</artifactId>
		<version>0.0.1-SNAPSHOT</version>
	</parent>
	<artifactId>datasource-dbcp</artifactId>

	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-dbcp2</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
		</dependency>
	</dependencies>
</project>
```

## 测试获取数据源配置信息

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class DateSourceTest {

	private final static Logger LOGGER = LoggerFactory.getLogger(DateSourceTest.class);
	
	@Autowired
	ApplicationContext applicationContext;

	@Autowired
	DataSource dataSource;

	@Autowired
	DataSourceProperties dataSourceProperties;

	@Test
	public void testDataSource() throws Exception {
		/**
		 * 可以直接使用@Autowired注解获取DataSource，也可以通过ApplicationContext上下文获取
		 */
		DataSource dataSource = applicationContext.getBean(DataSource.class);
		
		LOGGER.info("DataSourceProperties:{}", dataSourceProperties.getType().getName());
		LOGGER.info("DataSource:{}", dataSource.getClass().getName());
	}
}
```































