---
title: "只需三步实现Gateway结合Sentinel实现无侵入网关限流"
date: 2023-07-13T18:02:21+08:00
lastmod: 2023-07-13T18:02:21+08:00
author: ["Max"]
keywords: 
- 
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
    image: "posts/tech/tech1/1.png" #图片路径例如：posts/tech/123/123.png
    zoom: "50%"
    caption: "" #图片底部描述
    alt: ""
    relative: false
---


# 分享技术，用心生活

> 前言：本文基于您已有基础的可运行的微服务系统，使用了Sping Cloud Alibaba，Gateway,Nacos等；目标实现网关流控类型的限流。

----
顾名思义限流用于在高并发场景下限制请求流量的进入，保护系统不被冲垮。阿里巴巴的开源sentinel可以通过设置不同种类规则实现对不同的资源的保护。

- 资源：可以是任何东西；服务，方法，代码...
- 规则：流控规则、熔断降级规则、系统保护规则、热点规则、网关API分组规则、网关流控规则

本文使用的各版本对应关系如下（官方链接[：版本对应关系）](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)
````xml
<spring.boot>2.6.7</spring.boot>
<spring-cloud>2021.0.2</spring-cloud>
<spring-cloud-alibaba>2021.0.4.0</spring-cloud-alibaba>
````
----
### 本文目标

- 微服务整合sentinel
- 使用sentinel客户端生成网关限流规则，并持久化到nacos中
- 规则生效，产生限流效果

### 三步走战略

----
#### 1.整合sentinel
##### 1.1.服务端整合
###### 1.1.1.引入pom
```xml
<!--网关组件-->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!--sentinel限流-->
<dependency>
   <groupId>com.alibaba.cloud</groupId>
   <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<!-- SpringCloud Ailibaba Sentinel Gateway -->
<dependency>
   <groupId>com.alibaba.cloud</groupId>
   <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>

<!-- 通过nacos持久化流控规则 -->
<dependency>
   <groupId>com.alibaba.csp</groupId>
   <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```
配置Sentinel Starter，必须有`spring-cloud-alibaba-sentinel-gateway`，`spring-cloud-starter-gateway` 依赖来让 `spring-cloud-alibaba-sentinel-gateway` 模块里的 Spring Cloud Gateway 自动化配置类生效。

避坑点1：通过Spring Cloud Alibaba接入sentinel需要将`spring.cloud.sentinel.filter.enabled` 配置项置为 false（网关流控默认粒度为route和自定义API分组维度，不支持URL粒度）

避坑点2：通过Spring Cloud Alibaba Sentinel 数据源模块，网关流控规则数据源类型是 `gw-flow`而不是`flow`

###### 1.1.2.编写规则配置文件
在`gateway`module中新建application-sentinel.yaml文件

配置规则

```yaml
spring:
  cloud:
    sentinel:
      enabled: true
      datasource:
        ds1:
          nacos:
            server-addr: ${spring.cloud.nacos.config.server-addr}
            namespace: ${spring.cloud.nacos.config.namespace}
            data-id: ${spring.application.name}-gateway-flow-rules
            group-Id: SENTINEL_GROUP
            rule-type: gw-flow
            data-type: json
        ds2:
          nacos:
            server-addr: ${spring.cloud.nacos.config.server-addr}
            namespace: ${spring.cloud.nacos.config.namespace}
            data-id: ${spring.application.name}-gateway-api-rules
            group-Id: SENTINEL_GROUP
            rule-type: gw-api-group
            data-type: json
          ds3:
            nacos:
              server-addr: ${spring.cloud.nacos.config.server-addr}
              namespace: ${spring.cloud.nacos.config.namespace}
              data-id: ${spring.application.name}-degrade-rules
              group-Id: SENTINEL_GROUP
              rule-type: degrade
              data-type: json
          ds4:
            nacos:
              server-addr: ${spring.cloud.nacos.config.server-addr}
              namespace: ${spring.cloud.nacos.config.namespace}
              data-id: ${spring.application.name}-system-rules
              group-Id: SENTINEL_GROUP
              rule-type: system
              data-type: json
      log:
        dir: /home/omo/sentinel/logs
        # 启用向sentinel发送心跳
      eager: true
      scg:
        fallback:
          response-status: 429
          mode: response
          response-body: '{"code": 429,"message": "前方拥堵，请稍后再试！"}'
```

1.sentinel dashboard默认会以`spring.application.name`开头生成规则，ds1、ds2为本次测试的类型

2.data-type为读取数据类型，此处设置为json

3.最后的fallback配置了被限流后的自定义响应

###### 1.1.3.配置通信端口

服务和sentinel dashboard之间通信端口配置


```yaml
spring:
  cloud:
    sentinel:
      transport:
        # sentinel控制台地址
        dashboard: localhost:8081
        # 跟sentinel控制台交流的端口，随意指定一个未使用的端口即可，默认是8719
        port: 8719
        # Get heartbeat client local ip
        clientIp: 127.0.0.1
```


##### 1.2.sentinel dashboard整合
Sentinel 1.6.3 引入了网关流控控制台的支持，可以直接在 Sentinel 控制台上查看 API Gateway 实时的 route 和自定义 API 分组监控，管理网关规则和 API 分组配置

首先需要下载[控制台](https://github.com/alibaba/Sentinel/tree/master/sentinel-dashboard)工程代码，自行修改编译。也可以使用我改造后的工程，开箱即用：[sentinel-dashboard-1.8.6](https://github.com/max-holo/sentinel-dashboard1.8.6-nacos)，改造后的工程支持多种规则配置管理、规则推送（推送到nacos）

启动前配置nacos地址和命名空间（application.properties）

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/752e35ca51ed4c4d85ae98eefe5677c3~tplv-k3u1fbpfcp-watermark.image?)

启动参数配置（IDEA）


| <p align=center>参数</p> | <p align=center>作用</p> |
| --- | --- |
|-Dserver.port=8080|启动端口|
|  -Dcsp.sentinel.dashboard.server=localhost:8080|向 Sentinel 接入端指定控制台的地址  |
|-Dproject.name=sentinel-dashboard|向 Sentinel 指定应用名称|


```
-Dserver.port=8081 -Dcsp.sentinel.dashboard.server=localhost:8081 -Dproject.name=sentinel-dashboard
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9660e1071ea4daeb563796227fdfff4~tplv-k3u1fbpfcp-watermark.image?)

启动后效果

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81e13e019f7e4c799bb6cb8ac7995b13~tplv-k3u1fbpfcp-watermark.image?)


#### 2.push模式规则持久化
push模式流程

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5c0b1686b2b4e1f998788f44cc83962~tplv-k3u1fbpfcp-watermark.image?)

（下载sentinel dashboard官方版本时需要自行实现，懒人可直接使用上面我提供的改造后版本）

下面列出核心修改点

##### 2.1.修改nacosConfig类
```java
@Bean
public ConfigService nacosConfigService() throws Exception {
    Properties properties = new Properties();
    //nacos地址
    properties.put(PropertyKeyConst.SERVER_ADDR, nacosAddr);
    //namespace为空即为public
    properties.put(PropertyKeyConst.NAMESPACE, namespace);
    return ConfigFactory.createConfigService(properties);
}
```

以及对应规则类型的provider和publisher,以GatewayFlowRule举例：

##### 2.2.修改发布规则实现（publisher）
```java
@Component("gatewayFlowRuleNacosPublisher")
public class GatewayFlowRuleNacosPublisher implements DynamicRulePublisher<List<GatewayFlowRuleEntity>> {

    @Autowired
    private ConfigService configService;

    @Autowired
    private Converter<List<GatewayFlowRuleEntity>, String> converter;

    @Override
    public void publish(String app, List<GatewayFlowRuleEntity> rules) throws Exception {
        AssertUtil.notEmpty(app, "app name cannot be empty");
        if (rules == null) {
            return;
        }
        configService.publishConfig(app + NacosConfigUtil.GATEWAY_FLOW_DATA_ID_POSTFIX,
                NacosConfigUtil.GROUP_ID, converter.convert(rules));
    }
}
```

###### 2.3.修改获取规则实现（provider）
```java
@Component("gatewayFlowRuleNacosProvider")
public class GatewayFlowRuleNacosProvider implements DynamicRuleProvider<List<GatewayFlowRuleEntity>> {
    @Autowired
    private ConfigService configService;

    @Autowired
    private Converter<String, List<GatewayFlowRuleEntity>> converter;

    @Override
    public List<GatewayFlowRuleEntity> getRules(String appName) throws Exception {
        String rules = configService.getConfig(appName + NacosConfigUtil.GATEWAY_FLOW_DATA_ID_POSTFIX,
                NacosConfigUtil.GROUP_ID, 3000);
        if (StringUtil.isEmpty(rules)) {
            return new ArrayList<>();
        }
        return converter.convert(rules);
    }
}
```

##### 2.4.修改对应接口
（GatewayFlowRuleController中）

引入
```java
@Autowired
@Qualifier("gatewayFlowRuleNacosProvider")
private DynamicRuleProvider<List<GatewayFlowRuleEntity>> ruleProvider;

@Autowired
@Qualifier("gatewayFlowRuleNacosPublisher")
private DynamicRulePublisher<List<GatewayFlowRuleEntity>> rulePublisher;
```

再修改增删改查方法中的实现，具体细节参考：[sentinel-dashboard-1.8.6](https://github.com/max-holo/sentinel-dashboard1.8.6-nacos)

#### 3.配置规则，测试限流效果
##### 3.1 启动sentinel dashboard

启动步骤见上面阐述

##### 3.2 配置网关限流规则

###### 3.2.1 配置Route类型规则

针对user服务（为你的网关路由配置中的id）配置了QPS=5,流控方式为快速失败的规则

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd8ccd23cfad438fbe3f00b2e0c55e9c~tplv-k3u1fbpfcp-watermark.image?)

到nacos后台查看规则详情，已同步到nacos

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3cd1717f271447d8be2e75e44b07577~tplv-k3u1fbpfcp-watermark.image?)

同样在nacos中修改，dashboard也会同步更新，不再演示。
（规则属性含义参考：[限流字段](https://github.com/alibaba/Sentinel/wiki/%E7%BD%91%E5%85%B3%E9%99%90%E6%B5%81)）

###### 3.2.2.配置API分组类型规则

先配置API分组

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4403abaec27144a299bcef6dd6e15640~tplv-k3u1fbpfcp-watermark.image?)

再配置规则（下拉选择刚才配置的分组，同样设置QPS=5）

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/141bf2192cc1407aa1b594a3341fe7f4~tplv-k3u1fbpfcp-watermark.image?)


##### 3.3 测试限流效果

###### 3.3.1.Route类型限流

设置测试用例 设置循环10次

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7181d4ba26c54eea97feb1c0d5e3f960~tplv-k3u1fbpfcp-watermark.image?)

测试结果5次成功5次失败，符合预期，大功告成！

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf99c14a1179452994abd31c718600fd~tplv-k3u1fbpfcp-watermark.image?)

###### 3.3.2.API分组限流

可以看到测试报告中对/auth/oms/** 的路径匹配到了且限流可以生效

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6fe53f88840a45f9999b78915543ab89~tplv-k3u1fbpfcp-watermark.image?)

##### 4.扩展

- 网关流控原理，详见[官网](https://sentinelguard.io/zh-cn/docs/api-gateway-flow-control.html)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59e3eee9774943a790d241e9eed5af9a~tplv-k3u1fbpfcp-watermark.image?)

- 在使用SpringCloud Gateway和SpringCloud Sentinel时，官方不支持针对4xx,5xx响应码做异常统计熔断，可以自行实现，可参考[issue1842](https://github.com/alibaba/Sentinel/issues/1842) [issue2537](https://github.com/alibaba/Sentinel/issues/2537#issuecomment-1205963072)

----

记得一键三连哦~~





