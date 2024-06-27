---
title: "0成本自建docker镜像仓库，丝滑流畅拉取镜像"
date: 2024-06-27T17:44:03+08:00
lastmod: 2024-06-27T17:44:03+08:00
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
    image: "https://img.loadingspace.cn/blog-img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626172243.png" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---
### 分享技术，用心生活

---


# 前言
想必大家都听说了，国内镜像站几乎都用不了，对于开发者来说，无疑是个不好的消息。在docker pull时直接超时失败，拉取不下来镜像。那么有没有什么办法解决呢？有！还不止一种。

1. 通过docker配置文件配置可用的国内镜像源
2. 设置代理
3. 自建镜像仓库

方法1已经不太好使了，能找到可用的不多，有的还存在没有最新的镜像问题。

方法2可行，不过得要有科学上网的工具，再会一点配置代理的知识，操作起来稍稍复杂。

本文主要介绍第三种方法，上手快，简单，关键还0成本！

## 准备工作
1. 登录阿里云，[找到容器镜像服务](<https://cr.console.aliyun.com/>)，创建一个个人版实例。(第一次使用的话，会让设置访问密码。记住，后面会用)
2. 找到仓库管理-命名空间，新建一个命名空间且设置为公开

![20240626174632.png](https://img.loadingspace.cn/blog-img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626174632.png)
3.不要创建镜像仓库，回到访问凭证

可以看到，如下2个信息，一个是你的阿里云用户名，一个是你的仓库地址（后面有用）
```
sudo docker login --username=阿里云用户名 registry.cn-beijing.aliyuncs.com
```
## github配置
1. fork项目，地址：
[docker_image_pusher](https://github.com/tech-shrimp/docker_image_pusher)

(感谢tech-shrimp提供的工具)

2. 在fork后的项目中通过Settings-Secret and variables-Actions-New Repository secret路径，配置4个环境变量
- ALIYUN_NAME_SPACE-命名空间
- ALIYUN_REGISTRY_USER-阿里云用户名
- ALIYUN_REGISTRY_PASSWORD-访问密码
- ALIYUN_REGISTRY-仓库地址

<p align=center><img src="https://img.loadingspace.cn/blog-img/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626203514.png" alt="企业微信截图_20240626203514.png"  /></p>

3.配置要拉取的镜像
打开项目images.txt，每一行配置一个镜像，格式：name:tag 比如

![20240626213138.png](https://img.loadingspace.cn/blog-img/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626213138.png)

提交修改的文件，则会自动在`Actions`中创建一个workflow。等待片刻即可（1分钟左右）

![20240626212730.png](https://img.loadingspace.cn/blog-img/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626212730.png)

5.回到阿里云容器镜像服务控制台-镜像仓库

![20240626213555.png](https://img.loadingspace.cn/blog-img/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20240626213555.png)

可以看到镜像已成功拉取并同步到你自己的仓库中。
## 测试效果
我自己操作了下把nginx的镜像给拉了过来，找台服务器测试一下速度


![演示.gif](https://img.loadingspace.cn/blog-img/%E6%BC%94%E7%A4%BA.gif)
哈哈！这速度杠杠的吧! 用这个方式的好处是，借助github的action机制，直接从dockerhub上拉取任何你想要的镜像，也不用担心国内镜像站版本更新不及时的问题。再从自建的仓库中pull下来就可以啦！
如果有小伙伴没捣鼓成功的，可以留言给我。