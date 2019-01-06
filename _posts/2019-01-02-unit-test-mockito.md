---
layout: post
title:  "Mockito使用"
date:   2019-01-02 22:18:00
categories: Unit-Test
tags: Mockito
---

* content
{:toc}

Mockito是一个优秀的用于单元测试的mock框架,`目的和作用就是模拟一些在应用中不容易构造或者比较复杂的对象，从而把测试与测试边界以外的对象隔离开`.
[源码路径](https://github.com/mockito/mockito)
[【参考资料】](https://static.javadoc.io/org.mockito/mockito-core/2.23.4/org/mockito/Mockito.html)





## 原理、使用场景

通过定义基于方法的模拟调用规则模拟任何代码的调用过程替代真实代码执行

## Mockito,PowerMockito Api

- org.mockito.Mockito.mock: 根据给定的类或实例创建模拟对象实例.
- org.mockito.Mockito.when: 绑定模拟行为.
- org.mockito.Mockito.verify: 用于校验模拟行为的预期结果,比如调用次数与返回值.
- org.mockito.Matchers.any: 用于生成任意类型的对象,例如any(String.class).
- org.mockito.Mockito.times： 用于校验模拟方法被调用次数的匹配.
- org.mockito.Mockito.mockingDetails：查看是否为mock对象或者spy对象
- org.junit.runner.RunWith: Junit4之后开放给开发者自定义测试类运行器的注解.


- org.powermock.core.classloader.annotations.PrepareForTest(注解): 包裹上下文中的被模拟类和模拟类方法的调用者,提供一种标识给Powermock框架对其做字节码修改等处理.
- org.powermock.core.classloader.annotations.PowerMockIgnore(注解): 用于过滤由部分自定义类加载器或spi接口的类型的加载.
- org.powermock.api.mockito.PowerMockito.verifyStatic: 用于静态方法校验.
- org.powermock.api.mockito.PowerMockito.verifyPrivate: 用于私有方法校验.

## Mock编写流程

初始化Mock对象、定制规则、业务调研、验证执行过程或者结果


## 前期准备
```java
public interface PersonDao {
    boolean addPerson(Person person);

    void updatePerson(Person person);
}

public class PersonService {

    private PersonDao personDao;

    public PersonService(PersonDao personDao) {
        this.personDao = personDao;
    }

    public PersonService() {

    }

    public boolean addPerson(Person person) {
        return personDao.addPerson(person);
    }

    public void updPerson(Person person) {
        personDao.updatePerson(person);
    }
}
```
## 执行测试

### 验证行为

包括方法是否执行,调用次数等(times,never,atLeastOnce,atLeast,atMost),mock对象未进行交互(verifyZeroInteractions),未被验证到mock方法(verifyNoMoreInteractions)

```java
    
PersonDao mockPersonDao;

PersonService personService;

@Before
public void setUp() {
    mockPersonDao = mock(PersonDao.class);
    when(mockPersonDao.addPerson(any(Person.class))).thenReturn(true);
   personService = new PersonService(mockPersonDao);
}

@Test
public void testAddPerson() throws Exception {
    Person person = new Person("dlzp");
    boolean result = personService.addPerson(person);

    assertTrue(result);//校验结果

    //验证执行次数 verify(mockPersonDao) 默认校验执行过
    verify(mockPersonDao, times(1)).addPerson(person);
}

@Test
public void testInteraction() {
    List mock = mock(List.class);
    //未使用该mock对象进行交互
    verifyZeroInteractions(mock);

    mock.add("element");
    verify(mock).add("element");

    //查找是否有未验证的交互
    verifyNoMoreInteractions(mock);
}

```


### 模拟期望结果

mock方法执行后的返回值或者抛出的异常或者对未预设的调用返回默认期望值[Answer]或者调用真实的方法[thenCallRealMethod]

```java

/**
 * doReturn()|doThrow()| doAnswer()|doNothing()|doCallRealMethod()
 */
@Test(expected = RuntimeException.class)
public void testDoMethod() {

    Person person = new Person("dlzp");

    //无返回值的void方法
    doNothing()
            .doThrow(RuntimeException.class)
            .when(mockPersonDao).updatePerson(person);
    personService.updPerson(person);
    verify(mockPersonDao).updatePerson(person);

    personService.updPerson(person); //再次调用抛出异常
    
    
    @Test
    public void testReturnValueByParams() {
        Map<Object, Object> res = new HashMap<>(); 
        PersonService spy = spy(PersonService.class); //answer： AdditionalAnswers提供了许多默认的实现
        doAnswer(
                invocation -> {
                    Map map = (Map) invocation.getArguments()[0];
                    map.put("result", "fail");
                    return null;
                }
        ).when(spy).returnValueByParams(anyMap());
        spy.returnValueByParams(res);
        assertTrue(res.get("result").equals("fail"));

    }
}
/**
 * 校验抛出的异常
 */
@Test(expected = RuntimeException.class)
public void testException() {
    LinkedList mockedList = mock(LinkedList.class);
    doThrow(new RuntimeException()).when(mockedList).clear();
    //下面会抛RuntimeException
    mockedList.clear();
}

```


### 参数匹配

可以使用参数定制或者参数匹配器或者自定义参数匹配.[如果使用了参数匹配，那么所有的参数都必须通过matchers来匹配]

```java
/**
 * 参数匹配：一旦使用了参数匹配器来验证，那么所有参数都应该使用参数匹配,如：
 * verify(mock).someMethod(anyInt(), anyString(), eq("third argument"));
 * verify(mock).someMethod(anyInt(), anyString(), "third argument"); 此处将出现异常
 */
@Test
public void testMatchParams() {
    List<String> mockList = mock(ArrayList.class);
    //任意参数
    when(mockList.get(anyInt())).thenReturn("element");
    assertTrue(mockList.get(33).equals("element"));

    //校验任意参数是否执行get方法
    verify(mockList).get(anyInt());

    //校验确定参数是否执行get方法
    verify(mockList).get(eq(33));

    //校验从未执行，还有atLeastOnce、atLeast、atMost
    verify(mockList, never()).add("element");

}
其他内置匹配器如下：Mockito.eq、Mockito.matches、Mockito.any(anyBoolean , anyByte , anyShort , anyChar , anyInt ,anyLong , anyFloat , anyDouble , anyList , anyCollection , anyMap , anySet等等)、Mockito.isNull、Mockito.isNotNull、Mockito.endsWith、Mockito.isA。

/**
 * 自定义参数匹配器
 */
private class IsValid extends ArgumentMatcher {
    @Override
    public  boolean matches(Object argument){
        // matches
        return false;
    }

}

```

### 执行顺序

```java

/**
 * 校验执行的顺序
 */
@Test
public void testVerifyOrder() {
    List mock = mock(List.class);

    mock.add("first a");
    mock.add("second b");

    //创建inOrder
    InOrder inOrder = inOrder(mock);

    //验证调用次数，若是调换两句，将会出错，因为singleMock.add("was added first")是先调用的
    inOrder.verify(mock).add("first a");
    inOrder.verify(mock).add("second b");

    verifyNoMoreInteractions(mock);
}


```


### 注解
```java

@RunWith(MockitoJUnitRunner.class)
public class PersonServiceAnnotationTest {

    @Mock //@Spy
    PersonDao mockPersonDao;

    @InjectMocks
    PersonService personService; //此注解声明的变量需要用到mock对象，mockito会自动注入mock或spy成员

    @Before
    public void initMocks() {
        //启动注解，可以不设置
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testMock() throws Exception {

        when(mockPersonDao.addPerson(any())).thenReturn(true);
        Person person = new Person("dlzp");

        assertTrue(personService.addPerson(person));
        verify(mockPersonDao).addPerson(person);
    }
}
```
### 连续调用

```java
/**
 * 连续存根调用
 */
@Test
public void testStubbingConsecutiveCall() {
    CustomInterface mock = mock(CustomInterface.class);
    when(mock.someMethod("param"))
            .thenThrow(new RuntimeException("first throw exception")) //第一次调用抛出异常
            .thenReturn("dlzp_01") //第二次调用返回值
            .thenReturn("dlzp_02"); //第三次及后续调用返回值
    // 简写
    // when(mock.someMethod("param")).thenReturn("dlzp_01","dlzp_02");
    try {
        mock.someMethod("param");
    } catch (RuntimeException e) {
        assertTrue(e.getMessage().equals("first throw exception"));
    }

    assertTrue(mock.someMethod("param").equals("dlzp_01"));
    assertTrue(mock.someMethod("param").equals("dlzp_02"));

}
```
### spy监控真实对象
使用spy来监控真实的对象时，需要注意的是此时需要谨慎的使用when-then语句，而改用do-when语句

```java
/**
 * 监视真实的对象，当调用mock出的spyObject对象时，只有指定了具体的mock方法时才会执行mock的方法，否则执行原始方法
 */
@Test
public void testSpy() {
    List<String> strList = new ArrayList<>();
    List<String> spy = spy(strList);
    //设置mock的方法
    when(spy.size()).thenReturn(20);
    doReturn("dlzp_01").when(spy).get(1);
    spy.add("dlzp");//调用真实对象
    assertTrue(spy.get(0).equals("dlzp"));
    assertTrue(spy.get(1).equals("dlzp_01"));
    assertTrue(spy.size() == 20); //mock spy返回值

}
```
### ArgumentCaptor 捕获参数

```java
/**
 * 参数捕捉
 */
@Test
public void testCapturingArguments()  {
    List mockedList = mock(List.class);
    ArgumentCaptor<String> argument = ArgumentCaptor.forClass(String.class);
    mockedList.add("John");
    //验证后再捕捉参数
    verify(mockedList).add(argument.capture());
    //验证参数
    assertEquals("John", argument.getValue());
}
```
### 重置Mock

```java
@Test
public void testReset(){
    List mock = mock(List.class);
    when(mock.size()).thenReturn(10);
    mock.add(1);
    reset(mock);
}
```

### 超时

```java
/**
 * 超时验证
 */
@Test
public void testTimeout(){
    List mock = mock(List.class);
    doReturn("ele").when(mock).get(0);
    assertTrue(mock.get(0).equals("ele"));
    //测试程序将会在此阻塞100毫秒，timeout的时候再进行验证是否执行过get(0)方法
    verify(mock, timeout(100)).get(0);

}
```

## 完整实例

```java
public interface PersonDao {
    boolean addPerson(Person person);

    void updatePerson(Person person);
}
public class PersonService {

    private PersonDao personDao;

    public PersonService(PersonDao personDao) {
        this.personDao = personDao;
    }

    public PersonService() {

    }

    public boolean addPerson(Person person) {
        return personDao.addPerson(person);
    }

    public void updPerson(Person person) {
        personDao.updatePerson(person);
    }

    /**
     * 返回值通过参数返回
     * @param map
     */
    public void returnValueByParams(Map<Object,Object> map){
        map.put("result","success");
    }
}

@RunWith(MockitoJUnitRunner.class)
public class PersonServiceAnnotationTest {

    @Mock //@Spy
            PersonDao mockPersonDao;

    @InjectMocks
    PersonService personService; //此注解声明的变量需要用到mock对象，mockito会自动注入mock或spy成员

    @Before
    public void initMocks() {
        //启动注解，可以不设置
        MockitoAnnotations.initMocks(this);
    }

    @Test
    public void testMock() throws Exception {

        when(mockPersonDao.addPerson(any())).thenReturn(true);
        Person person = new Person("dlzp");

        assertTrue(personService.addPerson(person));
        verify(mockPersonDao).addPerson(person);
    }
}

/**
 * PersonService Tester.
 *
 * @author <dlzp>
 * @version 1.0
 * @since <pre>一月 1, 2019</pre>
 */
@RunWith(MockitoJUnitRunner.class)
public class PersonServiceTest {


    PersonDao mockPersonDao;

    PersonService personService;

    @Before
    public void setUp() {
        mockPersonDao = mock(PersonDao.class);
        when(mockPersonDao.addPerson(any(Person.class))).thenReturn(true);
        personService = new PersonService(mockPersonDao);
    }

    /**
     * Method: addPerson(Person person)
     */

    @Test
    public void testAddPerson() throws Exception {
        Person person = new Person("dlzp");
        boolean result = personService.addPerson(person);

        assertTrue(result);//校验结果

        //验证执行次数
        verify(mockPersonDao, times(2)).addPerson(person);
    }

    /**
     * 参数匹配：一旦使用了参数匹配器来验证，那么所有参数都应该使用参数匹配,如：
     * verify(mock).someMethod(anyInt(), anyString(), eq("third argument"));
     * verify(mock).someMethod(anyInt(), anyString(), "third argument"); 此处将出现异常
     */
    @Test
    public void testMatchParams() {
        List<String> mockList = mock(ArrayList.class);
        //任意参数
        when(mockList.get(anyInt())).thenReturn("element");
        assertTrue(mockList.get(33).equals("element"));

        //校验任意参数是否执行get方法
        verify(mockList).get(anyInt());

        //校验确定参数是否执行get方法
        verify(mockList).get(eq(33));

        //校验从未执行，还有atLeastOnce、atLeast、atMost
        verify(mockList, never()).add("element");

    }

    /**
     * 自定义参数匹配器
     */
    private class IsValid extends ArgumentMatcher {
        @Override
        public boolean matches(Object argument) {
            // matches
            return false;
        }

    }

    /**
     * 校验抛出的异常
     */
    @Test(expected = RuntimeException.class)
    public void testException() {
        LinkedList mockedList = mock(LinkedList.class);
        doThrow(new RuntimeException()).when(mockedList).clear();
        //下面会抛RuntimeException
        mockedList.clear();
    }

    /**
     * 校验执行的顺序
     */
    @Test
    public void testVerifyOrder() {
        List mock = mock(List.class);

        mock.add("first a");
        mock.add("second b");

        //创建inOrder
        InOrder inOrder = inOrder(mock);

        //验证调用次数，若是调换两句，将会出错，因为singleMock.add("was added first")是先调用的
        inOrder.verify(mock).add("first a");
        inOrder.verify(mock).add("second b");

        verifyNoMoreInteractions(mock);
    }


    @Test
    public void testInteraction() {
        List mock = mock(List.class);
        //未使用该mock对象进行交互
        verifyZeroInteractions(mock);

        mock.add("element");
        verify(mock).add("element");

        //查找是否有未验证的交互
        verifyNoMoreInteractions(mock);
    }

    /**
     * 连续存根调用
     */
    @Test
    public void testStubbingConsecutiveCall() {
        CustomInterface mock = mock(CustomInterface.class);
        when(mock.someMethod("param"))
                .thenThrow(new RuntimeException("first throw exception")) //第一次调用抛出异常
                .thenReturn("dlzp_01") //第二次调用返回值
                .thenReturn("dlzp_02"); //第三次及后续调用返回值
        // 简写
        // when(mock.someMethod("param")).thenReturn("dlzp_01","dlzp_02");
        try {
            mock.someMethod("param");
        } catch (RuntimeException e) {
            assertTrue(e.getMessage().equals("first throw exception"));
        }

        assertTrue(mock.someMethod("param").equals("dlzp_01"));
        assertTrue(mock.someMethod("param").equals("dlzp_02"));

    }


    private interface CustomInterface {
        String someMethod(String param);
    }

    /**
     * doReturn()|doThrow()| doAnswer()|doNothing()|doCallRealMethod()
     */
    @Test(expected = RuntimeException.class)
    public void testDoMethod() {

        Person person = new Person("dlzp");

        //无返回值的void方法
        doNothing()
                .doThrow(RuntimeException.class)
                .when(mockPersonDao).updatePerson(person);
        personService.updPerson(person);
        verify(mockPersonDao).updatePerson(person);

        personService.updPerson(person); //再次调用抛出异常
    }

    /**
     * 监视真实的对象，当调用mock出的spyObject对象时，只有指定了具体的mock方法时才会执行mock的方法，否则执行原始方法
     */
    @Test
    public void testSpy() {
        List<String> strList = new ArrayList<>();
        List<String> spy = spy(strList);
        //设置mock的方法
        when(spy.size()).thenReturn(20);
        doReturn("dlzp_01").when(spy).get(1);
        spy.add("dlzp");//调用真实对象
        assertTrue(spy.get(0).equals("dlzp"));
        assertTrue(spy.get(1).equals("dlzp_01"));
        assertTrue(spy.size() == 20); //mock spy返回值

    }

    /**
     * 参数捕捉
     */
    @Test
    public void testCapturingArguments() {
        List mockedList = mock(List.class);
        ArgumentCaptor<String> argument = ArgumentCaptor.forClass(String.class);
        mockedList.add("John");
        //验证后再捕捉参数
        verify(mockedList).add(argument.capture());
        //验证参数
        assertEquals("John", argument.getValue());
    }

    @Test
    public void testReset() {
        List mock = mock(List.class);
        when(mock.size()).thenReturn(10);
        mock.add(1);
        reset(mock);
    }

    /**
     * 超时验证
     */
    @Test
    public void testTimeout() {
        List mock = mock(List.class);
        doReturn("ele").when(mock).get(0);
        assertTrue(mock.get(0).equals("ele"));
        //测试程序将会在此阻塞100毫秒，timeout的时候再进行验证是否执行过get(0)方法
        verify(mock, timeout(100)).get(0);

    }

    @Test
    public void testReturnValueByParams() {
        Map<Object, Object> res = new HashMap<>();
        PersonService spy = spy(PersonService.class);
        doAnswer(
                invocation -> {
                    Map map = (Map) invocation.getArguments()[0];
                    map.put("result", "fail");
                    return null;
                }
        ).when(spy).returnValueByParams(res);
        spy.returnValueByParams(res);
        assertTrue(res.get("result").equals("fail"));

    }

}

```
