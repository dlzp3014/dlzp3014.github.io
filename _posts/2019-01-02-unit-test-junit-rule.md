---
layout: post
title:  "Junit使用Rule"
date:   2019-01-06 21:20:00
categories: Unit-Test
tags: Junit
---


* content
{:toc}

Rule是JUnit4中的新特性，它可以扩展JUnit的功能，灵活地`改变测试方法的行为`。JUnit中用@Rule和@ClassRule两个注解来实现Rule扩展，这两个注解需要放在实现了TestRule接口的成员变量（@Rule）或者静态变量（@ClassRule）上。@Rule和@ClassRule的不同点是，@Rule是方法级别的，每个测试方法执行时都会调用被注解的Rule，而@ClassRule是类级别的，在执行一个测试类的时候只会调用一次被注解的Rule




## JUnit中的内置Rule：
Junit中的Rule主要实现了TestRule接口,定义如下,内部仅有一个apply方法，用于修改方法运行的过程，并返回新的Statement

```java
/**
 * A TestRule is an alteration in how a test method, or set of test methods
 * is run and reported.  A {@link TestRule} may add additional checks that cause
 * a test that would otherwise fail to pass, or it may perform necessary setup or
 * cleanup for tests, or it may observe test execution to report it elsewhere.
 * {@link TestRule}s can do everything that could be done previously with
 * methods annotated with {@link org.junit.Before},
 * {@link org.junit.After}, {@link org.junit.BeforeClass}, or
 * {@link org.junit.AfterClass}, but they are more powerful, and more easily
 * shared
 * between projects and classes.
 *
 * The default JUnit test runners for suites and
 * individual test cases recognize {@link TestRule}s introduced in two different
 * ways.  {@link org.junit.Rule} annotates method-level 方法级别
 * {@link TestRule}s, and {@link org.junit.ClassRule} 类级别
 * annotates class-level {@link TestRule}s.  See Javadoc for those annotations
 * for more information.
 *
 * Multiple {@link TestRule}s can be applied to a test or suite execution. The
 * {@link Statement} that executes the method or suite is passed to each annotated
 * {@link org.junit.Rule} in turn, and each may return a substitute or modified
 * {@link Statement}, which is passed to the next {@link org.junit.Rule}, if any. For
 * examples of how this can be useful, see these provided TestRules,
 * or write your own:
 *  
 * <ul>
 *   <li>{@link ErrorCollector}: collect multiple errors in one test method</li>
 *   <li>{@link ExpectedException}: make flexible assertions about thrown exceptions</li>
 *   <li>{@link ExternalResource}: start and stop a server, for example</li>
 *   <li>{@link TemporaryFolder}: create fresh files, and delete after test</li>
 *   <li>{@link TestName}: remember the test name for use during the method</li>
 *   <li>{@link TestWatcher}: add logic at events during method execution</li>
 *   <li>{@link Timeout}: cause test to fail after a set time</li>
 *   <li>{@link Verifier}: fail test if object state ends up incorrect</li>
 * </ul>
 *
 * @since 4.9
 */
public interface TestRule {
    /**
     * Modifies the method-running {@link Statement} to implement this
     * test-running rule.
     *
     * @param base The {@link Statement} to be modified
     * @param description A {@link Description} of the test implemented in {@code base}
     * @return a new statement, which may be the same as {@code base},
     *         a wrapper around {@code base}, or a completely new Statement.
     */
    Statement apply(Statement base, Description description);
}
```

### TemporaryFolder Rule

使用这个Rule可以创建一些临时目录或者文件，在一个测试方法结束之后，系统会自动清空它们

```java
/**
 * 创建TemporaryFolder Rule:可以在构造方法上加入路径参数来指定临时目录，否则使用系统临时目录
 */
@Rule
public TemporaryFolder tempFolder = new TemporaryFolder();

@Test
public void testTempFolderRule() throws IOException {
    //在系统的临时目录下创建文件或者目录，当测试方法执行完毕自动删除
    tempFolder.newFile("test.txt");
    tempFolder.newFolder("test");
}


```

### ExternalResource Rule

ExternalResource 是TemporaryFolder的父类，主要用于在测试之前创建资源，并在测试完成后销毁

```java
File tempFile = null;
@Rule
public ExternalResource extResource = new ExternalResource() {
    //每个测试执行之前都会调用该方法创建一个临时文件
    @Override
    protected void before() throws Throwable {
        tempFile = File.createTempFile("test", ".txt");
    }

    //每个测试执行之后都会调用该方法删除临时文件
    @Override
    protected void after() {
        tempFile.delete();
    }
};

@Test
public void testExtResource() throws IOException {
    System.out.println(tempFile.getCanonicalPath()); //C:\Users\Administrator\AppData\Local\Temp\test7268759145628228861.txt

}
```

### ErrorCollector Rule

ErrorCollector允许收集多个错误，并在测试执行完后一次过显示出来

```java
@Rule  
public ErrorCollector errorCollector = new ErrorCollector();  
  
@Test  
public void testErrorCollector() {  
    errorCollector.addError(new Exception("error msg"));  
    errorCollector.addError(new Throwable("error msg"));  
}  
```

### Verifier Rule

Verifier是ErrorCollector的父类，可以在测试执行完成之后做一些校验，以验证测试结果是不是正确

```java
String result = null;  
  
@Rule  
public Verifier verifier = new Verifier() {  
    //当测试执行完之后会调用verify方法验证结果，抛出异常表明测试失败  
    @Override  
    protected void verify() throws Throwable {  
        if (!"success".equals(result)) {  
            throw new Exception("Test Fail.");  
        }  
    }  
};  
  
@Test  
public void testVerifier() {  
    result = "Fail";  
}  
```
### TestWatcher Rule

TestWatcher 定义了五个触发点，分别是测试成功，测试失败，测试开始，测试完成，测试跳过，能让在每个触发点执行自定义的逻辑

```java
@Rule  
public TestWatcher testWatcher = new TestWatcher() {  
    @Override  
    protected void succeeded(Description description) {  
        System.out.println(description.getDisplayName() + " Succeed");  
    }  
  
    @Override  
    protected void failed(Throwable e, Description description) {  
        System.out.println(description.getDisplayName() + " Fail");  
    }  
  
    @Override  
    protected void skipped(AssumptionViolatedException e, Description description) {  
        System.out.println(description.getDisplayName() + " Skipped");  
    }  
  
    @Override  
    protected void starting(Description description) {  
        System.out.println(description.getDisplayName() + " Started");  
    }  
  
    @Override  
    protected void finished(Description description) {  
        System.out.println(description.getDisplayName() + " finished");  
    }  
};  
  
@Test  
public void testTestWatcher() { 
    System.out.println("Test invoked");  
}  
```

### TestName Rule

```java
@Rule  
public TestName testName = new TestName();  
  
@Test  
public void testTestName() {  
    //打印出测试方法的名字testTestName  
    System.out.println(testName.getMethodName());  
}  
```

## 实现原理

在Junit4的默认Test Runner - org.junit.runners.BlockJUnit4ClassRunner中，有一个methodBlock方法

```java

/**
 * Returns a Statement that, when executed, either returns normally if 
 * {@code method} passes, or throws an exception if {@code method} fails.
 *
 * Here is an outline of the default implementation:
 *
 * <ul>
 * <li>Invoke {@code method} on the result of {@code createTest()}, and
 * throw any exceptions thrown by either operation.
 * <li>HOWEVER, if {@code method}'s {@code @Test} annotation has the {@code
 * expecting} attribute, return normally only if the previous step threw an
 * exception of the correct type, and throw an exception otherwise.
 * <li>HOWEVER, if {@code method}'s {@code @Test} annotation has the {@code
 * timeout} attribute, throw an exception if the previous step takes more
 * than the specified number of milliseconds.
 * <li>ALWAYS run all non-overridden {@code @Before} methods on this class
 * and superclasses before any of the previous steps; if any throws an
 * Exception, stop execution and pass the exception on.
 * <li>ALWAYS run all non-overridden {@code @After} methods on this class
 * and superclasses after any of the previous steps; all After methods are
 * always executed: exceptions thrown by previous steps are combined, if
 * necessary, with exceptions from After methods into a
 * {@link MultipleFailureException}.
 * <li>ALWAYS allow {@code @Rule} fields to modify the execution of the
 * above steps. A {@code Rule} may prevent all execution of the above steps,
 * or add additional behavior before and after, or modify thrown exceptions.
 * For more information, see {@link TestRule}
 * </ul>
 *
 * This can be overridden in subclasses, either by overriding this method,
 * or the implementations creating each sub-statement.
 */
protected Statement methodBlock(FrameworkMethod method) {
    Object test;
    try {
        test = new ReflectiveCallable() {
            @Override
            protected Object runReflectiveCall() throws Throwable {
                return createTest();
            }
        }.run();
    } catch (Throwable e) {
        return new Fail(e);
    }

    Statement statement = methodInvoker(method, test);
    statement = possiblyExpectingExceptions(method, test, statement);
    statement = withPotentialTimeout(method, test, statement);
    statement = withBefores(method, test, statement);
    statement = withAfters(method, test, statement);
    statement = withRules(method, test, statement); //执行rule规则
    return statement;
}
```
在JUnit执行每个测试方法之前，methodBlock方法都会被调用，用于把该测试包装成一个Statement。Statement代表一个具体的动作，例如测试方法的执行，Before方法的执行或者Rule的调用，类似于J2EE中的Filter，Statement也使用了责任链模式，将Statement层层包裹，就能形成一个完整的测试，JUnit最后会执行这个Statement。从上面代码可以看到，有以下内容被包装进Statement中：
- 测试方法的执行；
- 异常测试，对应于@Test(expected=XXX.class)；
- 超时测试，对应与@Test(timeout=XXX)；
- Before方法，对应于@Before注解的方法；
- After方法，对应于@After注解的方法；
- Rule的执行。

以withBefores(method, test, statement)为例获取Statement如下,内部重建了RunBefores Statement：
```java
protected Statement withBefores(FrameworkMethod method, Object target,
        Statement statement) {
    List<FrameworkMethod> befores = getTestClass().getAnnotatedMethods(
            Before.class);
    return befores.isEmpty() ? statement : new RunBefores(statement,
            befores, target);
}
```
RunBefores 继承了Statement 重写了 evaluate方法
```java
public class RunBefores extends Statement {
    private final Statement next;

    private final Object target;

    private final List<FrameworkMethod> befores;

    public RunBefores(Statement next, List<FrameworkMethod> befores, Object target) {
        this.next = next;
        this.befores = befores;
        this.target = target;
    }

    @Override
    public void evaluate() throws Throwable {
        for (FrameworkMethod before : befores) {
            before.invokeExplosively(target);
        }
        next.evaluate(); //执行下一个Statement 
    }
}
```
在evaluate中，所有Before方法会先被调用，因为Before方法必须要在测试执行之前调用，然后再执行fNext.evaluate()调用下一个Statement。

## 自定义Rule
主要实现TestRule接口，

```java
/* 
   用于循环执行测试的Rule，在构造函数中给定循环次数。 
 */  
public class LoopRule implements TestRule{  
    private int loopCount;  
  
    public LoopRule(int loopCount) {  
        this.loopCount = loopCount + 1;  
    }  
  
    @Override  
    public Statement apply(final Statement base, Description description) {  
        return new Statement() {  
            //在测试方法执行的前后分别打印消息  
            @Override  
            public void evaluate() throws Throwable {  
                for (int i = 1; i < loopCount; i++) {  
                    System.out.println("Loop " + i + " started!");  
                    base.evaluate();  
                    System.out.println("Loop "+ i + " finished!");  
                }  
            }  
        };  
    }  
}

@Rule  
public LoopRule loopRule = new LoopRule(3);  
  
@Test  
public void testLoopRule() {  
    System.out.println("Test invoked!");  
}  
```

    