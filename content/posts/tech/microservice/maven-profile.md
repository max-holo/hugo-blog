---
title: "微服务下，如何使用maven实现多环境配置？"
date: 2023-07-24T11:48:28+08:00
lastmod: 2023-07-24T11:48:28+08:00
author: ["Max"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- maven
- 多环境配置
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
    image: "https://img.loadingspace.cn/blog-img/maven-profile.png" 
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

### 分享技术，用心生活
---

>前言：很多项目在开发，提测，上线时都会提前手动改一些配置文件来适应对应环境，麻烦不说了，而且也容易出错；生产环境的配置也容易暴露。基于此，我们基于spring cloud alibaba架构下通过使用maven的profile来实现多环境切换的功能。

 ## 1 maven的profile介绍

 详细可查阅官网：[profile的描述](https://maven.apache.org/guides/introduction/introduction-to-profiles.html)

懒人版本可看下面的总结
  ### 1.1 在何处可以配置profile

  - 在项目的pom.xml：作用范围仅限当前项目
  - 在用户的setting.xml（(%USER_HOME%/.m2/settings.xml)）：作用范围仅限当前用户
  - 在全局的setting.xml（(${maven.home}/conf/settings.xml)）：作用范围为全部项目
  - ~~在项目的baseDir下的profiles.xml：maven3.0以上已废弃~~

  ### 1.2 激活profile的方式

  - 显式激活：直接用命令，比如`mvn groupId:artifactId:goal -P profile-1`
  - 隐式激活：标签`<activation>`可以配置根据jdk、操作系统、系统属性、文件是否存在等方式，在构建时自动检测这些配置

  ## 2 配置profile
  
  本文使用的是在项目的`pom.xml`配置，可以实现在IDEA中手动选择环境并构建。

   ### 2.1 配置各环境文件

   这里用网关module来操作

   在`resouces`目录下新建local,dev,test,prod文件，分别代表本地，开发，测试，生产环境。
   ![目录结构](https://img.loadingspace.cn/blog-img/mudules-menu.png)

   举例dev中的配置内容：主要是加载nacos开发环境的命名空间下的路由配置`gateway-route.yaml`，redis连接配置`gateway-redis.yaml`，mq连接配置`rabbitmq.yaml`

   你也可以把数据库连接配置也配置上，这样就达到了很好的屏蔽各种连接配置的暴露，尤其是账号密码。

   `bootstrap-dev.yaml`
   ```yml
   spring:
    cloud:
        nacos:
            config:
                extensionConfigs:
                -   dataId: gateway-route.yaml
                    refresh: true
                -   dataId: gateway-redis.yaml
                    refresh: true
                -   dataId: rabbitmq.yaml
                    refresh: true
                file-extension: yaml
                namespace: 2076a052-12fb-4ee5-ada1-c9bdcd2a0637
                server-addr: 192.168.0.246:8848
            discovery:
                namespace: 2076a052-12fb-4ee5-ada1-c9bdcd2a0637
                server-addr: 192.168.0.246:8848
   ```

配置`bootstrap.yaml`

   ```yml
   spring:
    profiles:
        active: @env@
   ```

- 如果你的项目引用了`spring-boot-starter-parent`，那么需要使用`@@`；反之使用`${}`;原因是starter在其中定义了`resource-delimiter`为`@`

![springboot-pom](https://img.loadingspace.cn/blog-img/pom-springboot.png)

本文用的是`@@`来读取变量，使用`spring.profiles.active`来使对应的配置文件生效


### 2.2 pom配置profiles

在`gateway`模块下找到上一级`support-module`，在其`pom.xml`中配置profile，共有4个，对应上面配置的4种环境文件。

1. 配置变量
```xml
<profiles>
		<profile>
			<id>dev</id>
			<properties>
				<env>dev</env>
			</properties>
			<!--默认环境-->
			<activation>
                <activeByDefault>true</activeByDefault>
			</activation>
		</profile>
		<profile>
			<id>local</id>
			<properties>
				<env>local</env>
			</properties>
		</profile>
		<profile>
			<id>test</id>
			<properties>
				<env>test</env>
			</properties>
		</profile>
		<profile>
			<id>prod</id>
			<properties>
				<env>prod</env>
			</properties>
		</profile>
	</profiles>
```

2. 配置资源目录

大家思考下，为什么会有这一步呢？

前面提到过`spring-boot-starter-parent`，它的pom下默认会有如下资源配置

![starter-resource](https://img.loadingspace.cn/blog-img/resouces-default.png)

简单来说，就是在构建时，需要copy`resources`目录下所有文件至`target`下，其中`<includes>`下文件需要进行变量替换（`filtering`为true）,关于`include` `exclude`深入研究可以查看[官网介绍](https://maven.apache.org/plugins/maven-resources-plugin/examples/include-exclude.html)

这个默认配置并不符合我们的要求，因为我们上面在gateway模块下创建了4个环境文件；如果按照默认配置的话我们虽然可以达到多环境的便捷使用效果，但是也同时copy了其他环境的文件。例如我们使用的是`dev`环境，同时`local` `test` `prod`下的文件也被构建在`target`下，这不是我们想要的，且仍然有生产环境的配置泄露的风险。

所以，就有了这一步，我们需要自己配置资源目录，来覆盖默认的。

通过配置'`bootstrap-${env}.yaml`来指定激活相对应的环境配置，这样就不会出现额外的环境配置了

通过上面可以知道gateway模块resources目录下存在国际化文件`properties`和`application` `bootstrap`文件且都需要构建到`target`，所以我们配置如下：
![resources](https://img.loadingspace.cn/blog-img/gateway-resource.png)

## 3 演示效果
配置完毕后，我们就可以在IDEA的maven面板处看到我们配置的profiles
1. 在IDEA中选择`local`环境

![profiles](https://img.loadingspace.cn/blog-img/idea-profiles.png)

2. 启动gateway

从启动日志中可以看到已经加载到正确的文件

![run-local](https://img.loadingspace.cn/blog-img/run-local.png)

也可以在启动类`Application`中打印环境变量
```java
	public static void main(String[] args) {
		ApplicationContext applicationContext = SpringApplication.run(GatewayApplication.class, args);
		Environment env = applicationContext.getEnvironment();
		System.out.println("----run env："+env.getProperty("spring.profiles.active")+"----");
	}
```

再选择`dev`环境，启动

![run-dev](https://img.loadingspace.cn/blog-img/run-dev.png)

第一个预期目标达成！


3. 校验是否只加载了local配置文件

切换回local，在maven面板找到`m`图标,左侧编写命令，右侧选择gateway
![mvn-package](https://img.loadingspace.cn/blog-img/maven-package.png)
回车执行，等待打包完成，查看`target`目录

![target-mulu](https://img.loadingspace.cn/blog-img/target-mulu.png)

完美的只加载了`bootstrap-local.yaml`文件，第二个预期目标达成！

最后，我们在切换环境时，最好点一下maven面板中的`reload`按钮，防止切换不生效。

