---
title: "微服务下，如何实现多设备同时登录或强制下线？"
date: 2023-07-21T14:03:02+08:00
lastmod: 2023-07-21T14:03:02+08:00
author: ["Max"]
keywords: 
- 多设备登录
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 多设备登录
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
    image: "https://img.loadingspace.cn/blog-img/cover1.png" 
    zoom: 100% # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---
### 分享技术，用心生活
---
>前言：你有没有遇到过这样的需求，产品要求实现同一个用户根据后台设置允许同时登录，或者不准同时登录时，需要强制踢下线前一个的场景。本文将带领大家实现一个简单的这种场景需求。
---

先来看一下简单的时序图，方便后续理解。

![时序图](https://img.loadingspace.cn/blog-img/sequence.png)

首先我们需要有一个后台设置开关来控制允不允许用户多设备同时登录的功能（没有也无妨，假定允许），其次在登录后，需要保存用户的userId-token的关系缓存。再回头看上面的时序图，是不是已经能理解实现的原理了。

如果你的架构是微服务，那么可以使用redis来存登录关系缓存，单体架构可以直接存session即可。本文是微服务架构，所以采用的是redis。

本文的前提都是基于同一个用户的情况下，下文不再赘述。

## 1 构造登录缓存关系

如果要实现同一用户多设备同时登录，那必然需要在session（微服务中可以用redis做session共享）中能找到用户的每一个登录状态，如果只是简单的缓存用户信息是实现不了的，登录时那就必须要有一个唯一值token，这样每次登录token不一样，但是指向的用户是同一个。

![user](https://img.loadingspace.cn/blog-img/user_redis.png)

`usertoken`中维护的是前缀:用户id,这里不需要维护多个，因为用的reids的hash数据类型，多个登录时，添加新行即可；`user`部分，这里维护的是多个，即登录一次就有一条记录；因为根据业务需要，后续需要从缓存中获取用户其他信息。

- 允许多设备同时登录：`usertoken`只有1条，`user`可能会有多条
- 不允许多设备同时登录(有则强制下线)：`usertoken`只有1条，`user`只有1条

```java
    /**
     * 登录成功后缓存用户信息
     *
     * @param req
     * @return
     */
    public void cacheUserInfo(CacheUserInfoReqDTO req) {
        // 1、缓存用户信息
        cacheUser(req);
        cacheAuth(req.getUid(), req.getRoles(), req.getPermissions());

        // 2、更新token与userId关系
        String userTokenRelationKey = RedisKeyHelper.getUserTokenRelationKey(req.getEntId() + SymbolConstant.COLON + req.getUid());
        redisAdapter.set(userTokenRelationKey, req.getToken(), RedisTtl.USER_LOGIN_SUCCESS);
    }
```

## 2 过滤器配置

登录鉴权部分和用户登录状态上下文不在本文范围内，此处忽略

登录成功后，每一个请求到达过滤器时，通过请求header中的`token`来获取登录信息;因为我们存的缓存key前缀都包含userId,所以要想得到用户信息，需要使用到redis的scan命令来获取。（**scan最好配置count来限制，保证性能**）

```java
@Override
protected Mono<Void> innerFilter(ServerWebExchange exchange, WebFilterChain chain) {
        String token = filterContext.getToken();
        if (StringUtils.isBlank(token)) {
            throw new DataValidateException(GatewayReturnCodes.TOKEN_MISSING);
        }

        // scan获取user的key
        String userKey = "";
        Set<String> scan = redisAdapter.scan(GatewayRedisKeyPrefix.USER_KEY.getKey() + "*" + token);
        if (scan.isEmpty()) {
            throw new DataValidateException(GatewayReturnCodes.TOKEN_EXPIRED_LOGIN_SUCCESS);
        }
        userKey = scan.iterator().next();
        
        MyUser myUser = (MyUser) redisAdapter.get(userKey);
        if (myUser == null) {
            throw new BusinessException(GatewayReturnCodes.TOKEN_EXPIRED_LOGIN_SUCCESS);
        }

        // 将用户信息塞入http header
        // do something...
        return chain.filter(exchange.mutate().request(newServerHttpRequest).build());
    }
```

这样保证即使有多设备同时登录，也能获取到登录信息和上下文。

## 3 如何做强制下线呢？

其实也很简单，在登录前可以通过`AOP`方式做校验，如果已登录了，那么这里就清除session或用户缓存，再继续进行正常登录即可。再简单一点可以直接在登录service中添加校验

核心逻辑

```java
 String userTokenRelationKey = RedisKeyHelper.getUserTokenRelationKey(req.getEntId() + SymbolConstant.COLON + userEntList.get(0).getUserId());
            String redisToken = (String) redisAdapter.get(userTokenRelationKey);
if (StringUtils.isNotEmpty(redisToken) && !redisToken.equals(token)) {
        throw new BusinessException(UserReturnCodes.MULTI_DEVICE_LOGIN);
            }
```
这里用于判断是否已有登录，并返回给前端提示。用于前端其他业务处理
如果不需要给前端提示，不用返回前端，直接进行清除session或用户缓存逻辑。

```java
 String userTokenRelationKey = RedisKeyHelper.getUserTokenRelationKey(req.getEntId() + SymbolConstant.COLON + userEntity.getId());
    // 获取当前已用户的登录token
    String redisToken = (String) redisAdapter.get(userTokenRelationKey);
    // 踢下线之前全部登录
    Response<Void> exitLoginResponse = gatewayRpc.allExit(ExitLoginReqDTO.builder().token(redisToken).userId(userEntity.getId()).build());
```

## 4 演示

- 演示强制下线

这里我用用户id为4做演示

先正常第一次登录,提示成功，并且redis中有1条`user`记录
![redis_succ](https://img.loadingspace.cn/blog-img/login_success.png)
![redis_succ](https://img.loadingspace.cn/blog-img/redis_login.png)

再次登录,我这里是返回给前端处理了，所以会有提示信息。

![login](https://img.loadingspace.cn/blog-img/code2.png)

前端效果

![message](https://img.loadingspace.cn/blog-img/login_message.png)

最后，扩展一下，如果要实现登录后强制修改默认密码、登录时间段限制等场景，你会怎么实现呢？






