---
title: "巧用设计模式实现多种登录方式"
date: 2023-07-19T13:58:07+08:00
lastmod: 2023-07-19T13:58:07+08:00
author: ["Max"]
keywords: 
- 登录
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 设计模式
- 登录
description: ""
weight: 2
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
    image: "https://img.loadingspace.cn/blog-img/strategy.png"
    zoom: # 图片大小，例如填写 100% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---
### 分享技术，用心生活


>前言：本文通过策略模式来实现客户端用户的不同登录方式。

---

## 1 概念

先抛个砖，什么是策略模式？和模板模式有什么区别？别急着看，先自己思考一下...


深入讨论的话，本文的篇幅不够，可以网上搜索，资料很多。
简单来说策略是针对一个行为有多套实现，模板是针对一个算法中步骤，其中不同场景的步骤有多种实现；

针对本文的登录场景，可以知道 登录=行为，手机验证码登录、账号密码登录、人脸识别登录等=实现；所以这部分可以通过策略模式来实现。那么这里可不可以使用模板模式呢？答案是 可以的！

不管是什么登录方式，核心步骤是相同的，通过用户的输入信息 大概分为以下步骤：
- 获取账号信息
- 校验账号状态
- 记录登录状态

其中第一步获取账号信息，就可以根据不同的登录方式，来各自实现不同的逻辑（通过手机号获取、通过账号名获取等）。这三步就可以使用模板模式，其中第一步有不同实现，二三步相同，做封装处理。策略+模板，高兴坏老板！

峰回路转，理论上是可以二者结合，但是在本场景下会大大增加代码维护的复杂度，反而适得其反。所以本文只采取策略模式搞定登录这个行为。

`设计模式不要为了使用而使用`需要因地制宜，实事求是。

## 2 策略模式实现

1. UML结构图
    ![](https://img.loadingspace.cn/blog-img/loginuml.png)

    我们可以看到策略模式有三种角色
    - 抽象策略：LoginStrategy 定义行为
    - 具体策略：AccountLoginProcessor、MobileLoginProcessor、FaceLoginProcessor 行为实现
    - 环境角色：AbstractLogin 用于暴露给客户端引用

2. 定义抽象策略

    这里我们定义一个行为动作：登录，并返回用户信息

```java
/**
 * 登录策略
 *
 * @author max
 * @date 2023/02/16 17:48
 */
public interface LoginStrategy {

    /**
     * 通用登录接口
     *
     * @param loginReqVO
     * @return UserEntity
     * @throws BusinessException;
     * @author max
     * @date 2023/02/16 17:42
     */
    UserEntity login(LoginReqVO loginReqVO) throws BusinessException;
```

3. 定义环境角色

    - 首先定义了一个ConcurrentHashMap 用来维护不同的登录方式
    - 通过前端传递的登录方式，调用不同的`loginProcessor`

```java
@Component
public abstract class AbstractLogin implements LoginStrategy {

    @Resource
    private IRedisAdapter redisAdapter;

    // 类初始化时，将子类对象存储在map中，key：登录类型
    public static ConcurrentHashMap<Integer, AbstractLogin> loginStrategyMap = new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        loginStrategyMap.put(getLoginType(), this);
    }


    /**
     * 通用登录接口
     *
     * @param loginReqVO
     * @return LoginRespVO
     * @author max
     * @date 2023/02/16 17:42
     */
    @Override
    public UserEntity login(LoginReqVO loginReqVO) throws BusinessException {
        // 1.其他公共登录处理逻辑可放在此
        // 2.调用相应类型的登录执行器
        return loginProcessor(loginReqVO);
    }

    /**
     * 在子类中声明登录类型
     *
     * @return int
     * @author max
     * @date 2023/02/16 18:00
     */
    protected abstract Integer getLoginType();

    /**
     * 登录执行器
     *
     * @param loginReqVO
     * @return UserEntity
     * @author max
     * @date 2023/02/16 18:00
     */
    protected abstract UserEntity loginProcessor(LoginReqVO loginReqVO);
}
```

4. 具体策略实现
 
账号密码登录
```java
    /**
 * 账号密码登录
 *
 * @author max
 * @date 2023/02/16 18:13
 **/
@Slf4j
@Service
@RequiredArgsConstructor
@DS(DataSourceName.BASIC_USER_MASTER)
public class AccountLoginProcessor extends AbstractLogin {

    private final UserInfoApiBiz userInfoApiBiz;

    @Override
    protected Integer getLoginType() {
        return LoginTypeEnum.ACCOUNT_LOGIN.getValue();

    }

    /**
     * 登录执行器
     *
     * @return UserEntity
     * @author max
     * @date 2023/02/16 18:00
     */
    @Override
    @DS(DataSourceName.BASIC_USER_SLAVE)
    protected UserEntity loginProcessor(LoginReqVO req) {
        // 校验账户名
        UserEntity userEntity = userInfoApiBiz.selectByLoginName(req.getUsername(),req.getEntId());
        if (Objects.isNull(userEntity)) {
            throw new BusinessException(UserReturnCodes.ACCOUNT_NOT_EXIST);
        }
        // 校验密码
        String securePassword = PasswordUtil.secure(req.getPassword(), "");
        if (!Objects.equals(securePassword, userEntity.getPassword().getValue())) {
            throw new BusinessException(UserReturnCodes.USERNAME_OR_PASSWORD_ERROR);
        }
        return userEntity;
    }

}
```
手机号登录

```java
/**
 * 手机号登录
 *
 * @author max
 * @date 2023/02/16 18:13
 **/
@Slf4j
@Service
@RequiredArgsConstructor
@DS(DataSourceName.BASIC_USER_MASTER)
public class MobileLoginProcessor extends AbstractLogin {

    private final UserInfoApiBiz userInfoApiBiz;

    @Override
    protected Integer getLoginType() {
        return LoginTypeEnum.MOBILE_LOGIN.getValue();

    }

    /**
     * 登录执行器
     *
     * @return int
     * @author max
     * @date 2023/02/16 18:00
     */
    @Override
    @DS(DataSourceName.BASIC_USER_SLAVE)
    protected UserEntity loginProcessor(LoginReqVO req) {
        // 校验手机号
        UserEntity userEntity = userInfoApiBiz.selectByMobile(req.getMobile(), req.getEntId());
        if (Objects.isNull(userEntity)) {
            throw new BusinessException(UserReturnCodes.ACCOUNT_NOT_EXIST);
        }
        return userEntity;
    }

}
```

人脸登录不再赘述，通过以上可以很清晰的举一反三。同时也可以看到我们只是在这里面通过不同的参数去获取到用户信息，还没有登录。这里是做了一个优化，把登录提到外面service中了（也可以放在`AbstractLogin`的第二步之后），减少冗余代码；继续往下看。

客户端引用


这里简化了下写法，方便大家理解

```java
@DS(DataSourceName.BASIC_USER_SLAVE)
public void login(LoginReqVO req) {
    AbstractLogin abstractLogin = AbstractLogin.loginStrategyMap.get(req.getLoginType().getValue());
    if (Objects.isNull(abstractLogin)) {
        throw new BusinessException(UserReturnCodes.LOGIN_TYPE_NOT_SUPPORT);
    }

    // 1.登录执行器
    UserEntity userEntity = abstractLogin.login(req);
    // 2.登录成功,记录登录状态
    // do something...

}
```

## 3 实战演示效果

用手机号登录的方式请求一下看看(我这里设置的登录类型的值为1)

![请求](https://img.loadingspace.cn/blog-img/requestDemo.png)

在service中断点查看，根据前端传的登录方式，我们可以看出map中已经匹配到了具体的登录策略实现

![步骤一](https://img.loadingspace.cn/blog-img/login-step1.png)

那么能否找到具体的实现呢？看下图断点可知，已经找到响应的手机号登录实现类

![步骤二](https://img.loadingspace.cn/blog-img/login-step2.png)

返回到service,可知手机号登录实现类已经正常返回了登录用户信息

![步骤三](https://img.loadingspace.cn/blog-img/login-step3.png)

至此，一个简单的结合策略模式实现不同登录方式的代码已经写完，符合开闭原则。如果有新的登录类型，那么增加一个子类即可。

如果你还有一些登录时的其他规则校验，可以在`AbstractLogin`的步骤1中添加，比如游戏的未成年保护，仅限制在某某时间段才可以登录、不允许多终端登录、仅PC端登录等场景。但是，如果规则太多了的话，写在一起也是很难看的

关注不迷路，下一篇教你如何优雅的实现多终端登录、登录设备的校验、登录后强制修改默认密码等常用场景。















