---
title: "巧用网关白名单实现接口免鉴权"
date: 2023-07-27T15:03:03+08:00
lastmod: 2023-07-27T15:03:03+08:00
author: ["Max"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 网关
- 白名单
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
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

### 分享技术，用心生活

>场景描述：一般系统中提供的接口都是经过统一配置鉴权的，比如不登录不能访问。但是，一些接口是需要开放给客户用的，我称作open API。那么这时候你不能要求客户先登录你的接口再去调用吧。那么，这时候就可以通过网关白名单来实现免鉴权

先说思路：
1. 配置网关白名单列表
2. 编写鉴权过滤器
3. 过滤器中读取白名单
4. 业务处理

简单的时序图

![时序图](https://img.loadingspace.cn/blog-img/filter_uml2.png)

**注：** 如果使用的是网关过滤器，在校验后应该再次过滤器，也就是经过2次；注意区别（网关过滤器具有前置pre、后置post两次过滤，细节不在此处详细探讨）。

过滤器普遍用于处理拦截，校验，改写，日志等场景；通过白名单来控制鉴权，正契合过滤器的作用。

## 1. 配置网关白名单

在你的本地的配置文件或者是nacos的配置文件中新增以下配置

可以配置url全路径，也可以配置前缀路径
```yml
gateway:
  whitelist:
    - /user/api/userInfo/query
    - /open/oss/upload
    - /open/vod/api
```

## 2. 过滤器配置

过滤器你可以选择用spring的`WebFilter`，如果你的系统集成了gateway也可以使用网关过滤器，然后自定义过滤器实现`GlobalFilter `

### 2.1. WebFilter实现

```java
@Component
@RequiredArgsConstructor
public class AuthFilter implements WebFilter, Ordered {

    private final GateWayProperties gateWayProperties;

    private static final AntPathMatcher pathMatcher = new AntPathMatcher();

    @Override
    public int getOrder() {
        return 1;
    }

    @Override
    protected Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String urlMethod = request.getURI().getPath() + request.getMethodValue();

        // 白名单匹配，直接放行
        for (String pattern : gateWayProperties.getWhitelist()) {
            if (pathMatcher.matchStart(pattern, urlMethod)) {
                return chain.filter(exchange);
            }
        }
       // 未匹配到
       // 鉴权逻辑，此处省略....
    }

}
```

### 2.2. 网关GlobalFilter实现

```java
@Component
@RequiredArgsConstructor
public class AuthFilter implements GlobalFilter, Ordered {

    private final GateWayProperties gateWayProperties;

    private static final AntPathMatcher pathMatcher = new AntPathMatcher();


    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String urlMethod = request.getURI().getPath() + request.getMethodValue();

        // 白名单匹配，直接放行
        for (String pattern : gateWayProperties.getWhitelist()) {
            if (pathMatcher.matchStart(pattern, urlMethod)) {
                return chain.filter(exchange);
            }
        }
        // 未匹配到，忽略鉴权逻辑，直接设置401
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        return exchange.getResponse().setComplete();
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

### 2.3. 读取白名单配置

`GateWayProperties`中的prefix和配置文件中的名称一致

```java
@Getter
@Setter
@ToString
@ConfigurationProperties(prefix = "gateway")
public class GateWayProperties implements Serializable {

    private static final long serialVersionUID = 1L;

    private List<String> whitelist;

}
```

## 3. 演示效果

使用上面配置的查询用户信息接口`/user/api/userInfo/query`做演示

### 3.1. 在白名单内

- `WebFilter`效果

查看断点`gateWayProperties`中白名单列表已获取到，且比对结果为true

![WebFilter](https://img.loadingspace.cn/blog-img/webfilter.png)

- `GlobalFilter`效果

查看断点`gateWayProperties`中白名单列表也已获取到，且比对结果为true

![GlobalFilter](https://img.loadingspace.cn/blog-img/gatewayfilter.png)

查询结果：已获取到用户信息

![结果](https://img.loadingspace.cn/blog-img/filter_success.png)

### 3.2. 不在白名单内

我们把接口`/user/api/userInfo/query`从白名单中删除，用网关过滤器演示。

查看断点`gateWayProperties`中白名单列表已没有查询用户接口，且返回了401

![GlobalFilter](https://img.loadingspace.cn/blog-img/filter_fail.png)

查询结果：http状态码是我们设置的401

![结果](https://img.loadingspace.cn/blog-img/fail-401.png)

当然，使用白名单也不仅仅局限于对外开放接口这个场景，也不仅仅局限于使用在鉴权过滤器上。这里只是一个抛砖引玉。实际需求可以结合自己的业务场景，使用不同的过滤器。



