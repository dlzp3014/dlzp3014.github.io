---
layout: post
title:  "PowerMock使用"
date: 2019-01-01 22:44:00
categories: Unit-Test
tags: PowerMock
---

* content
{:toc}

PowerMock是一个扩展了jMock 、EasyMock、Mockitomock等mock框架，功能更加强大的单元测试框架，以弥补其他框架不能mock静态、final、私有方法等功能。PowerMock使用一个自定义类加载器和字节码操作来模拟静态方法，构造函数，final类和方法，私有方法，去除静态初始化器等等。通过使用自定义的类加载器，简化采用的IDE或持续集成服务器，不需要做任何改变。PowerMock旨在用少量的方法和注解扩展现有的API来实现额外的功能。目前PowerMock支持EasyMock和Mockito。[PowerMock源码地址及相关文档](https://github.com/powermock/powermock)，[PowerMock Examples](https://github.com/powermock/powermock-examples-maven)
 



 
 
 
## PowerMock实现原理

- 当某个测试方法被注解@PrepareForTest标注以后，在运行测试用例时，会创建一个新的org.powermock.core.classloader.MockClassLoader实例，然后加载该测试用例使用到的类(系统类除外)
-  PowerMock会根据你的mock要求，去修改写在注解@PrepareForTest里的class文件（当前测试类会自动加入注解中），以满足特殊的mock需求。例如：去除final方法的final标识，在静态方法的最前面加入自己的虚拟实现等
-如果需要mock的是系统类的final方法和静态方法，PowerMock不会直接修改系统类的class文件，而是修改调用系统类的class文件，以满足mock需求

## PowerMock使用步骤

- 通过PowerMockito.mock(Class<T> clazz)可以创建一个mock的对象实例
对Mock的对象，使用类似PowerMockito.when(mockObject.someMethod(someArg)).thenReturn(resultYouExpected)的语法，可以mock对象的行为的返回值。需要注意的是：mock的对象，`没有调用when设置过的方法，在测试时调用，返回的都是对应返回类型的默认值`。

- 测试时，希望对行为进行验证，可以使用类似PowerMockito.verify(mockObject[,times(int)]).someMethod(somgArgs)的语法，验证mock的对象，方法是否执行过，执行过几次。

- 特别的，对于希望调用后什么都不做的，或者抛出异常的，可以使用PowerMockito.doNothing().when(...)...、doThrow(Throwable).when(...)...的语法

- 如果一个对象，只希望mock它的部分方法，而其他方法希望和真实对象的行为一眼，可以使用PowerMockito.spy(Class<T> clazz)代替PowerMockito.mock(Class<T> clazz)方法，其后的设置依旧，这时，没有通过when设置过的方法，测试调用时，行为和真实对象一样

- 通配符使用
测试时对于传入的参数或者传出的参数并不关心，这时可以使用通配符。此时可以通过PowerMockito.anyInt()、PowerMockito.anyString、甚至PowerMockito.any(Class<T> clazz)……来表示任意值；需要注意的是：`如果一个方法中，有一个地方使用了通配符，其他参数也都要使用通配符，对于特定的参数，不能直接指定，而需要使用PowerMockito.eq(someArg)来通配这个参数`

- private/static/final method:
在测试类上使用注解@RunWith(PowerMockRunner.class)、@PrepareForTest(targetClassNeedMock.class);
对于静态方法：@PrepareForTest的class是静态方法所在类，对这个类使用PowerMockito.mock(Class<T> hasStaticMethodClazz),之后对静态方法使用PowerMockito.when(mockObject.someStaticMethod(...))...mock，调用静态方法的被测试类即可被mock;对于final方法：与静态方法类似，不过@PrepareForTest的class是final方法所在类
;对于私有方法：PowerMockito提供了对私有方法的mock，使用PowerMockito.when(mockObject,methodName:String,someArgs:Object)...来mock私有方法，这时@PrepareForTest的class是私有方法所在类 

【Note】:@PrepareForTest的class，它总代表需要特殊处理的类，而不一定是被测试类。需要特殊处理的类可能只是被测试类的一个成员变量，它可以在mock后自动注入被测试类中

- 修改私有属性
在Mockito1.x和2.x，修改私有属性的方法略有不同：
1.x的使用Whitebox.setInternalState(mockObject,"attributeName",mockAttributeBean)的方式可以设置私有属性；2.x的使用FieldSetter.setField(mockObject,originAttrubuteBean,mockAttributeBean)的方式来设置,这个时候，需要先获得原先mock对象内的成员变量(:Field),所以便利性降低了

- 阻止静态方法的执行 @SupressStaticInitializationFor
一些静态块的代码，可能测试的时候不想让它们在类加载的时候执行，这个时候，可以使用@SuppressStaticInitializationFor("会执行静态方法的全限定名")来阻止执行

- 跳过父构造函数
`需要继承一些来自第三方库的类，而有些类在单元测试中无法成功的构造`,此时可以使用:
MemberModifier.suppress(constructor(Parent.class));将父类的构造器跳过

- 自身构造器:Whitebox.newInstance()来跳过构造函数而直接创建实例

- 跳过自身方法调用:MemberModifier.suppress(method(Clazz.class, "methodName"));

- 跳过属性赋值属性：
```java  
  private String privateFiled = Object.getStaticMethod();
  suppress(field(Clazz.class, "privateFiled"));
```    
- 模拟策略:@MockPolicy

@MockPolicy(Slf4jMockPolicy.class)

- 测试监听器:@PowerMockListener

@PowerMockListener(AnnotationEnabler.class)
@PowerMockListener(FieldDefaulter.class)

- 与SpringJunit一起使用:Spring工程中需要使用@Runwith(SpringJUnit4ClassRunner.class),但是PowerMockito也需要Runwith注解。这种情况，PowerMockito提供了新的注解[@PowerMockRunnerDelegate](https://github.com/powermock/powermock/wiki/JUnit_Delegating_Runner),来允许另一个Junit Runner运行:

```java

@Component
public class MyBean {

    public Message generateMessage() {
        final long id = IdGenerator.generateNewId();
        return new Message(id, "My bean message");
    }
}
public class IdGenerator {

	/**
	 * @return A new ID based on the current time.
	 */
	public static long generateNewId() {
		return System.currentTimeMillis();
	}
}
@RunWith(PowerMockRunner.class)
@PowerMockRunnerDelegate(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:/example-context.xml")
@PrepareForTest(IdGenerator.class)
public class SpringExampleTest {

    @Autowired
    private MyBean myBean;

    @Test
    public void mockStaticMethod() throws Exception {
        // Given
        final long expectedId = 2L;
        mockStatic(IdGenerator.class);
        when(IdGenerator.generateNewId()).thenReturn(expectedId);

        // When
        final Message message = myBean.generateMessage();

        // Then
        assertEquals(expectedId, message.getId());
        assertEquals("My bean message", message.getContent());
    }
}
```
`对于@Spy和@InjectMocks标注的对象，它们同时还可以使用@Autowired标注，这时，没有被mock的部分，将会从Spring注入进来，这些对象表现如同真实对象一样`


## 使用PowerMockito时常用的注解

- @RunWith(PowerMockRunner.class)
- @PrepareForTest( { NeedMockClass.class }): 需要使用PowerMock强大功能（Mock静态、final、私有方法等）的时候，就需要加注解@PrepareForTest

## PowerMockito核心方法

- PowerMockito.spy 构造Mock对象
- PowerMockito.mock
- PowerMockito.mockStatic 静态方法
- PowerMockito.doReturn mock返回值
- PowerMockito.doNothing 无返回值
- PowerMockito.when.withArguments.thenReturn/thenCallRealMethod mock方法的链式调用
- PowerMockito.whenNew 构造对象
- PowerMockito.verifyPrivate 验证私有方法
- PowerMockito.verifyStatic 验证静态方法

## Maven 配置

```java
<properties>
    <junit.version>4.12</junit.version>
    <powermock.version>1.7.1</powermock.version>
</properties>
<groupId>tech.dlzp.java.code</groupId>
<artifactId>unit-test</artifactId>
<dependencies>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>${junit.version}</version>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-module-junit4</artifactId>
        <version>${powermock.version}</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.powermock</groupId>
        <artifactId>powermock-api-mockito</artifactId>
        <version>${powermock.version}</version>
        <scope>test</scope>
    </dependency>
```
## PowerMock基本用法

### 静态方法

```java
public class HelperClass {

    public static boolean staticMethod() {
        // do something
        return false;
    }


    public final boolean finalMethod() {
        // do something
        return false;
    }
}

@Test
public void testMockStaticMethod_01() {
    mockStatic(HelperClass.class);
    when(HelperClass.staticMethod()).thenReturn(true);
    assertTrue(new PowserMockDemo().mockStaticMethod());

}

@Test
public void testMockStaticMethod_02() throws Exception {
    spy(HelperClass.class);
    doReturn(true).when(HelperClass.class, "staticMethod");//可以使用此中方式mock私有静态方法，返回值为VOID时可使用doNothing
    assertTrue(new PowserMockDemo().mockStaticMethod());

}
//final method
@Test
public void testMockFinalMethod() throws Exception {
    HelperClass helperClass = mock(HelperClass.class);
    when(helperClass.finalMethod()).thenReturn(true); 
    assertTrue(new PowserMockDemo().mockFinalMethod(helperClass));
}

```

### 私有方法
```java
private boolean privateMethod(){
    return false;
}

public boolean mockPrivateMethod(){
    return privateMethod();
}

@Test
public void testMockPrivateMethod_01() {
    PowserMockDemo powserMockDemo = new PowserMockDemo();
    MemberModifier.stub(MemberMatcher.method(PowserMockDemo.class,
            "privateMethod")).toReturn(true);
    assertTrue(powserMockDemo.mockPrivateMethod());
}

@Test
public void testMockPrivateMethod_02() throws Exception {
    PowserMockDemo mock = mock(PowserMockDemo.class);

    when(mock.mockPrivateMethod()).thenCallRealMethod();//mock出的对象调用真实的方法
    when(mock, "privateMethod").thenReturn(true);
    assertTrue(mock.mockPrivateMethod());
}

@Test
public void testMockPrivateMethod_03() throws Exception {
    //使用spy创建mock对象后，只有设置了mock的方法才会调用mock行为
    PowserMockDemo powserMockDemo = spy(new PowserMockDemo());
    when(powserMockDemo, "privateMethod").thenReturn(true);//也使用使用deReturn(...).when(...)
    assertTrue(powserMockDemo.mockPrivateMethod());
}

```

### 构造方法

```java
/**
 * @throws Exception
 */
@Test
public void testMockConstructor() throws Exception {
    File file = PowerMockito.mock(File.class);
    whenNew(File.class).withArguments("filePath").thenReturn(file); //私有构造方法：PowerMockito.constructor(PowerMockito.class).newInstance(new Object[]{});
    PowserMockDemo powserMockDemo = new PowserMockDemo();
    when(file.exists()).thenReturn(false);
    assertFalse(powserMockDemo.mockConstructor("filePath"));
}
```
### Mock方法中的参数

```java
/**
 * Method: mockMethodParams(File file)
 */
@Test
public void testMockMethodParams() {
    //mock出入参File对象
    File file = mock(File.class);
    PowserMockDemo powserMockDemo = new PowserMockDemo();
    when(file.exists()).thenReturn(true);
    assertTrue(powserMockDemo.mockMethodParams(file));
}
```
### Mock私有变量

```java
@Test
public void testPrivateField_01() throws IllegalAccessException {
    //mock private field
    PowserMockDemo powserMockDemo = new PowserMockDemo();

    MemberModifier
            .field(PowserMockDemo.class, "mockPrivateField").set(
            powserMockDemo, true);


    assertTrue(powserMockDemo.getMockPrivateField());
}

@Test
public void testPrivateField_02() throws Exception {
    //mock private field
    PowserMockDemo powserMockDemo = new PowserMockDemo();
    Whitebox.setInternalState(powserMockDemo, "mockPrivateField", true);
    assertTrue(powserMockDemo.getMockPrivateField());
    //verifyPrivate(Mockito.times(1));
}
```
### 验证

```java
@Test
public void testVerifyStatic() throws Exception {

    //检查某个私有方法调用的次数
    PowserMockDemo spy = spy(new PowserMockDemo());
    spy.mockPrivateMethod();
    verifyPrivate(spy, Mockito.times(1)).invoke("privateMethod");


    //检查静态方法调用的次数:先调用对应的静态方法，再启用静态检查，并定义规则，再次调用对应的静态方法，查看是否是通过校验的
    spy(HelperClass.class);
    HelperClass.staticMethod();
    verifyStatic(HelperClass.class, Mockito.times(1));
    HelperClass.staticMethod();
}
```

## 完整Demo

### 业务的端：
```java
public class HelperClass {

    public static boolean staticMethod() {
        // do something
        return false;
    }


    public final boolean finalMethod() {
        // do something
        return false;
    }
}

public class PowserMockDemo {

    private boolean mockPrivateField;

    public boolean getMockPrivateField() {
        return mockPrivateField;
    }

    public boolean mockMethodParams(File file) {

        return file.exists();
    }

    public boolean mockConstructor(String filePath) {

        return new File(filePath).exists();
    }

    public boolean mockFinalMethod(HelperClass helperClass) {
        return helperClass.finalMethod();
    }

    public boolean mockStaticMethod(){
        return  HelperClass.staticMethod();
    }

    private boolean privateMethod(){
        return false;
    }

    public boolean mockPrivateMethod(){
        return privateMethod();
    }

}
```

### UT by PowerMockito

```java
/**
 * PowserMockDemo Tester.
 *
 * @author <dlzp>
 * @version 1.0
 * @since <pre>一月 1, 2019</pre>
 */
@RunWith(PowerMockRunner.class)
@PrepareForTest({HelperClass.class, PowserMockDemo.class})
public class PowserMockDemoTest {

    @Before
    public void before() throws Exception {
    }

    @After
    public void after() throws Exception {
    }

    /**
     * Method: mockMethodParams(File file)
     */
    @Test
    public void testMockMethodParams() {
        //mock出入参File对象
        File file = mock(File.class);
        PowserMockDemo powserMockDemo = new PowserMockDemo();
        when(file.exists()).thenReturn(true);
        assertTrue(powserMockDemo.mockMethodParams(file));
    }

    /**
     * @throws Exception
     */
    @Test
    public void testMockConstructor() throws Exception {
        File file = PowerMockito.mock(File.class);
        whenNew(File.class).withArguments("filePath").thenReturn(file);
        PowserMockDemo powserMockDemo = new PowserMockDemo();
        when(file.exists()).thenReturn(false);
        assertFalse(powserMockDemo.mockConstructor("filePath"));
    }

    @Test
    public void testMockFinalMethod() throws Exception {
        HelperClass helperClass = mock(HelperClass.class);
        when(helperClass.finalMethod()).thenReturn(true);
        assertTrue(new PowserMockDemo().mockFinalMethod(helperClass));
    }

    @Test
    public void testMockStaticMethod_01() {
        mockStatic(HelperClass.class);
        when(HelperClass.staticMethod()).thenReturn(true);
        assertTrue(new PowserMockDemo().mockStaticMethod());

    }

    @Test
    public void testMockStaticMethod_02() throws Exception {
        spy(HelperClass.class);
        doReturn(true).when(HelperClass.class, "staticMethod");
        assertTrue(new PowserMockDemo().mockStaticMethod());

    }

    @Test
    public void testMockPrivateMethod_01() {
        PowserMockDemo powserMockDemo = new PowserMockDemo();
        MemberModifier.stub(MemberMatcher.method(PowserMockDemo.class,
                "privateMethod")).toReturn(true);
        assertTrue(powserMockDemo.mockPrivateMethod());
    }

    @Test
    public void testMockPrivateMethod_02() throws Exception {
        PowserMockDemo mock = mock(PowserMockDemo.class);

        when(mock.mockPrivateMethod()).thenCallRealMethod();//mock出的对象调用真实的方法
        when(mock, "privateMethod").thenReturn(true);
        assertTrue(mock.mockPrivateMethod());
    }

    @Test
    public void testMockPrivateMethod_03() throws Exception {
        //使用spy创建mock对象后，只有设置了mock的方法才会调用mock行为
        PowserMockDemo powserMockDemo = spy(new PowserMockDemo());
        when(powserMockDemo, "privateMethod").thenReturn(true);//也使用使用deReturn(...).when(...)
        assertTrue(powserMockDemo.mockPrivateMethod());
    }

    @Test
    public void testPrivateField_01() throws IllegalAccessException {
        //mock private field
        PowserMockDemo powserMockDemo = new PowserMockDemo();

        MemberModifier
                .field(PowserMockDemo.class, "mockPrivateField").set(
                powserMockDemo, true);


        assertTrue(powserMockDemo.getMockPrivateField());
    }

    @Test
    public void testPrivateField_02() throws Exception {
        //mock private field
        PowserMockDemo powserMockDemo = new PowserMockDemo();
        Whitebox.setInternalState(powserMockDemo, "mockPrivateField", true);
        assertTrue(powserMockDemo.getMockPrivateField());
        //verifyPrivate(Mockito.times(1));
    }


    @Test
    public void testVerifyStatic() throws Exception {

        //检查某个私有方法调用的次数
        PowserMockDemo spy = spy(new PowserMockDemo());
        spy.mockPrivateMethod();
        verifyPrivate(spy, Mockito.times(1)).invoke("privateMethod");


        //检查静态方法调用的次数:先调用对应的静态方法，再启用静态检查，并定义规则，再次调用对应的静态方法，查看是否是通过校验的
        spy(HelperClass.class);
        HelperClass.staticMethod();
        verifyStatic(HelperClass.class, Mockito.times(1));
        HelperClass.staticMethod();
    }
    
    /**
    * mock ExecutorService
    */
    @Test
    public void testExecutorService() {
        ExecutorService executor = PowerMockito.mock(ThreadPoolExecutor.class);
        PowerMockito.doAnswer(invocationOnMock -> {
            Runnable command = invocationOnMock.getArgumentAt(0,Runnable.class);
            command.run();
            return null;
        }).when(executor).execute(Mockito.any(Runnable.class));

    }
}
```
### 备注
Mock是CI利器，能够在测试阶段最大程度减少各开发团队之间的耦合，但它并非万能，毕竟实际上线时业务模块之间必然是真实调用，所以它并不能替代联调测试！
