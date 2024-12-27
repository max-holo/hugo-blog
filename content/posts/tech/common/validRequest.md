---
title: "服务端参数校验之分组校验、嵌套校验"
date: 2023-08-20T17:22:50+08:00
lastmod: 2023-08-20T17:22:50+08:00
author: ["Max"]
keywords: 
- 参数校验
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://img.loadingspace.cn/blog-img/validrequest.png" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

### 分享技术，用心生活
--- 

日常开发中，免不了需要对请求参数进行校验，诸如判空，长度，正则，集合等，复杂一点的请求参数可能会包含嵌套，分组校验。
我们由简入深开始，一文搞定参数校验！

## 1. 简单判空

- GET请求，字符串类型参数：使用@NotBlank注解 `@NotBlank String mobile`
- GET请求，int,long byte等类型参数：使用@NotNull注解 `@NotNull Integer userNum`
- POST请求，以body中参数为json为例：使用@Valid注解 `@Valid UserReq userReq`,UserReq中字段使用`@NotBlank` 或`@NotNull`

以下均为POST请求，以body中参数类型为json举例
## 2. 参数长度校验
```java
    @Size(min=5,max = 20)
    private String nickName;
```
长度5-20之间

## 3. 正则校验
```java
    @Pattern(regexp = "^\\d{15}|\\d{18}$")
    private String idCard;
```
15或18位数字

## 4. 集合校验

```java
    @NotEmpty()
    private List<User> users;
```
集合不能为空

## 5.分组校验

分组校验需要先定义好分组，比如
![分组](https://img.loadingspace.cn/blog-img/validgroup.png)

举例，定义一个`AddView`类
```java
public interface AddView {
}
```

针对上面的`users`集合使用分组
```java
public class UserReq {
    private static final long serialVersionUID = 1L;

    /**
     * id
     */
    @NotNull()
    private Long id;

    /**
     * 用户集合
     */
    @NotEmpty(groups = {AddView.class, UpdateView.class})
    private List<User> users;
}
```
通过配置`groups`使`users`在新增和修改的时候才会校验。

在请求方法上设置` @Validated`,在修改`users`时，校验参数不能为空集合
```java
    @PostMapping("users")
    public Response<Void> users(@RequestBody @Validated({UpdateView.class}) UserReq userReq) {
        // to do something
        return RespUtil.success();
    }
```

## 6. 嵌套校验
同样的我们用`users`举例；如果我们想要对`users`中某个字段也进行校验，那么怎么实现呢？
也很简单，只需要再加一个`@Valid`
```java
public class UserReq {
    private static final long serialVersionUID = 1L;

    /**
     * id
     */
    @NotNull()
    private Long id;

    /**
     * 用户集合
     */
    @NotEmpty(groups = {AddView.class, UpdateView.class})
    @Valid
    private List<User> users;
}
```
`User`内字段设置
```java
public class User {
    private static final long serialVersionUID = 1L;

    /**
     * id
     */
    @NotNull()
    private Long userId;

    /**
     * 姓名
     */
    @NotBlank
    private String name;
}
```
这样就实现了参数的嵌套校验+分组校验的组合。
当然，在包`javax.validation.constraints`下还有很多其他注解来选择支持不同场景的需要，比如`@DecimalMax` `@DecimalMin` `@Email` `@Max` `@Min` `Negative`等，这里仅列举常用的几个起到抛砖引玉的作用。

## 7. 请求header参数校验
有时候我们不单单需要校验body中参数，还有可能需要校验header中参数，比如常见的token啊、timestamp啊等等。
那就可以利用spring提供的`@RequestHeader`来实现，用法也很简单
```java
    @PostMapping("login")
    public Response<Void> login(@RequestBody @Valid LoginReq loginReq,
                                @NotBlank @RequestHeader("token") String token) {
        // to do something
        return RespUtil.success();
```
这里我们就实现了对header参数`token`的判空处理。

后记：参数校验场景各种各样，对于这些简单的使用，掌握好了还是能够覆盖大部分需求的；常用的必须掌握，不常用的我们需要知道，万一哪天遇到了，我们就知道在哪里去查现成的轮子可以使用；当然，对于复杂的参数校验，有可能需要您自定义注解实现，或者通过过滤器等方式实现。不必拘泥于固定形式。一切以结果为导向。






