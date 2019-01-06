---
layout: post
title:  "PowerMock中Answer与doAnswer的使用"
date:   2019-01-05 17:50:00
categories: Unit-Test
tags: PowerMock
---
* content
{:toc}

在某些边缘的情况下不可能通过简单地通过PowerMockito.when().thenReturn()模拟，如根据入参的不同返回不同的结果集，这时就需要使用Answer接口:
Answer接口指定执行的action和返回值。 






Answer的参数是InvocationOnMock的实例，支持：

- callRealMethod()：调用真正的方法

- getArguments()：获取所有参数

- getMethod()：返回mock实例调用的方法

- getMock()：获取mock实例

```java

public String doAnser(String str){
    // do something
    return str;
}

public class UserAnswerTest {


    @Test
    public void testAnswer() {
        HelperClass mock= PowerMockito.spy(new HelperClass());

        doAnswer(answer->
                answer.getArgumentAt(0,String.class)
                        .equalsIgnoreCase("ok")? "success": "fail")
                        //或者直接返回default value return success;此时可以使用defaultAnswer;
                .when(mock).doAnser(Mockito.anyString());
       Assert.assertEquals(mock.doAnser("ok"),"success");
       Assert.assertEquals(mock.doAnser("no"),"fail");
    }

}
```