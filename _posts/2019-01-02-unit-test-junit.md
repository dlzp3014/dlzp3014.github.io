---
layout: post
title:  "Junit使用"
date:   2019-01-02 00:57:00
categories: Unit-Test
tags: Junit
---


* content
{:toc}

[ JUnit 4 ](https://junit.org/junit4/) -- [javadoc](https://junit.org/junit4/javadoc/latest/index.html)

[ JUnit 5 ](https://junit.org/junit5/) -- [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/)





## 使用Junit时注意事项

- 测试方法上面必须使用@Test注解进行修饰。
 
- 测试方法必须使用public void 进行修饰，不能带有任何参数。
 
- 新建一个源代码目录用来存放测试代码。
 
- 测试类的包应该与被测试类的包保持一致。

- 测试单元中的每一个方法必须独立测试，每个测试方法之间不能有依赖。
 
- 测试类使用Test做为类名的后缀（非必要）。

- 测试方法使用test作为方法名的前缀（非必要）。

## 错误解析

- Failure:一般是单元测试使用的断言方法判断失败引起，说明预期结果和程序运行结果不一致。

- error:是有代码异常引起的，产生于测试代码本身中的Bug。

- 测试用例是不是用来证明你是对的，而是用来证明你没有错。


## 测试套件

@RunWith()：
- Suite.class:测试运行器Suite.class，@Suite.SuiteClasses：将需要运行的测试类放入Suite.SuiteClasses({})的数组中,以执行多个测试类
- Parameterized.class:测试数据,@Parameters

## Junit常用注解

### @BeforeClass
所修饰的方法在所有方法加载前执行，而且是静态的在类加载后就会执行该方法 在内存中只有一份实例，适合用来加载配置文件。

### @AfterClass
所修饰的方法在所有方法执行完毕之后执行，通常用来进行资源清理，例如关闭数据库连接。

### @Before和@After在每个测试方法执行前都会执行一次。

### @Test
- (excepted=XX.class) 异常测试。

```java
@Test(expected = RuntimeException.class,)
public void testException(){
    throw new RuntimeException("error msg");
}
```
当前测试仅能测试出方法按照预期抛出异常，如果代码里面不只一个地方抛出RuntimeException（只是包含的信息不一样），还是无法分辨具体的异常信息。这种情况可以使用JUnit的ExpectedException Rule来解决。
```java
 @Test
public void testExpectedExceptionRule(){
    //期待抛出RuntimeException
    expectedException.expect(RuntimeException.class);
    //期待抛出的异常信息中包含"Access Denied"字符串
    expectedException.expectMessage(CoreMatchers.containsString("Access Denied"));
    //当然也可以直接传入字符串，表示期待的异常信息（完全匹配）
    //expectedException.expectMessage("Access Denied!");
}
```
- (timeout=毫秒) 允许程序运行的时间。

### @Ignore 

所修饰的方法被测试器忽略。

### @RunWith 

可以修改测试运行器 org.junit.runner.Runner

### 执行顺序
- 一个测试类单元测试的执行顺序为:
@BeforeClass -> @Before -> @Test -> @After -> @AfterClass
- 每一个测试方法的调用顺序为
@Before –> @Test –> @After

## JUnit常用断言
    JUnit提供了一些辅助函数，用来确定被测试的方法是否按照预期的效果正常工作，通常，把这些辅助函数称为断言

- assertArrayEquals(expecteds,actuals):检查数组是否相等
- assertEquals(expecteds,actuals):检查对象是否相等
- assertNotEquals(expecteds,actuals):检查对象不相等
- assertNotNull(Object object):对象不为为null检查
- assertNull(Object object):对象为null检查
- assertTrue(boolean condition):true
- assertFalse(boolean condition):false 
- assertThat(T actual, Matcher<? super T> matcher):检查结果是否满足指定的条件


## Junit Demo

```java
public class JunitTest {

    /**
     * @BeforeClass修饰的方法会在所有方法被调用前被执行，该方法是静态的，
     * 所以当测试类被加载后接着就会运行它,而且在内存中它只会存在一份实例，它比较适合加载配置文件
     */
    @BeforeClass
    public static void initResource(){
        System.out.println("init resource...");
    }

    /**
     *  @AfterClass所修饰的方法通常用来对资源的清理，如关闭数据库的连接
     */
    @AfterClass
    public static void releaseResource(){
        System.out.println("release resource...");

    }

    @Before
    public void setUp(){
        System.out.println("Called before each test method");
    }

    /**
     * @Test注解方法中抛出了异常时，所有的@After注解方法依然会被执行
     */
    @After
    public void setDown(){
        System.out.println("Called after each test method");
    }

    @Test
    public void testMethod(){
        System.out.println("execute testMethod");
        List<String> list = Arrays.asList("a");
        Assert.assertTrue(list.get(0).equals("a"));
    }

    @Test
    public void testMethod02(){
        System.out.println("execute testMethod02");
        List<String> list = Arrays.asList("a");
        Assert.assertTrue(list.get(0).equals("a"));
    }
}

console:

init resource...
Called before each test method
execute testMethod02
Called after each test method
Called before each test method
execute testMethod
Called after each test method
release resource...

```