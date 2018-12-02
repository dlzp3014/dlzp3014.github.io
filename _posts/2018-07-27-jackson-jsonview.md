---
layout: post
title:  "Jackson @JsonView 注解使用"
date:   2018-08-21 23:49:00
categories: jackson
tags: jackson SpringMVC
---

* content
{:toc}

@JsonView是jackson中的一个注解,SpringMVC也支持这个注解,主要用于控制输入输出后的json.如根据不同的RESTfull接口返回同一个对象的不同属性值




`JsonView的使用步骤:`

### 使用接口声明多个视图,使用@JsonView在对应的属性上选择指定的接口视图进行标注

```java
/**
 * @Description: @JsonView 注解返回特定的属性值
 * @Author: dlzp
 * @Date: 2018/8/22 0022 23:05
 */
public class User {

    //user列表展示数据
    public interface UserListView{

    }
    //完整视图接口 允许返回用户名密码属性，使用继承获取列表数据
    public interface UserDetailView extends UserListView{

    }

    @JsonView(UserListView.class)
    private Integer userId;
    @JsonView(UserListView.class)

    private String userName;
    @JsonView(UserDetailView.class)
    private String password;

    public User(String userName, String password) {
        this.userName = userName;
        this.password = password;
    }

    public Integer getUserId() {
        return userId;
    }

    public void setUserId(Integer userId) {
        this.userId = userId;
    }

    public User() {
    }

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

}

```

### Controller中使用@JsonView设置定义的视图
```java
    /**
    * @Description: @JsonView  列表视图
    * @param:
    * @return:
    */
    @GetMapping("listUser")
    @JsonView(User.UserListView.class)
    public List<User> listUser(){
        return Arrays.asList(new User("dlzp","password"));
    }

    /**
    * @Description: @JsonView  详情视图
    * @param: [userId]
    * @return: tech.dlzp.java.dto.User
    */
    @GetMapping("/{userId}/userDetail")
    @JsonView(User.UserDetailView.class)
    public User userDetail(@PathVariable  String userId){
        return new User("dlzp","password");
    }
```

### 测试

```java
ObjectMapper objectMapper = new ObjectMapper();
//创建对象
User user = new User("dlzp","password");
//序列化
ByteArrayOutputStream bos = new ByteArrayOutputStream();
objectMapper.writerWithView(User.UserDetailView.class).writeValue(bos, user);
System.out.println(bos.toString());

bos.reset();
objectMapper.writerWithView(User.UserListView.class).writeValue(bos, user);
System.out.println(bos.toString());

```


    
