---
title: "5分钟搭建免费图床应用到个人博客"
date: 2023-07-17T16:20:23+08:00
lastmod: 2023-07-17T16:20:23+08:00
author: ["Max"]
keywords: 
- 
categories: 
- 
tags: 
- tools
description: ""
weight:
slug: ""
draft: false # 是否为草稿
comments: true
reward: true # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
    image: "https://img.loadingspace.cn/blog-img/tuchuang.png"
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

> 前言：本文将实战性的实现使用七牛云搭建免费图床；并应用到自己的博客。

***

图床：用于保存上传图片的地方（服务器），并且可以提供外链到互联网访问。

有的小伙伴也可以直接使用网络上提供的免费图床，也是可以的。不过别人的总不如自己的好（稳定，可靠）。教员说 自己动手，丰衣足食嘛。

# 1.开通账号

在[七牛云官网](https://www.qiniu.com/)注册账号后，即可拥有永久免费10G的存储空间，对于个人使用基本上足够。当然也可以使用阿里OSS、腾讯COS基本上都不贵，方法类似。

##### 1.1 创建空间bucket

在菜单：对象存储kodo->空间管理->新建空间，新建的空间访问控制选择公开；这样后续博客使用时才有权限访问（可以通过加白名单方式做防盗链，见下文）
![空间bucket](https://img.loadingspace.cn/blog-img/qiniu-bucket.png)

再新建一个目录

![目录](https://img.loadingspace.cn/blog-img/qiniu-mulu.png)

创建空间后，会给一个默认的域名供测试（外链访问），有效期30天

![image.png](https://img.loadingspace.cn/blog-img/qiniu-domain.png)

也可以设置自定义域名，下文会讲到。

##### 1.2 获取密钥

右上角头像->密钥管理，会出现一对AK/SK，用于对接七牛云。

# 2.使用picGo对接七牛云

##### 2.1 下载picGo

下载地址[picGo](https://github.com/Molunerfinn/PicGo/releases/tag/v2.3.1)
打开picGo，选择七牛云进行配置（用到上面的密钥）
![picgo](https://img.loadingspace.cn/blog-img/picgo.png)

*   bucket：即为创建的空间
*   访问网址：可以用七牛云提供的测试域名，这里我绑定了自定义域名
*   存储路径：即为创建的目录

设置后，picGo自动帮我们对接好七牛云。可直接在上传区操作，上传完毕后到目录下查看即可。

到此，我们的免费图床基本搭建好，下面要结合博客使用，我们总不能使用测试域名吧，30天到期后，图直接全部挂掉，直接GG！接下来是设置自定义域名

# 3.设置自定义域名

前提是，你已经有一个已备案的域名。如果没有可直接结束此文并点赞收藏了\~（〒▽〒 狗头.jpg）

##### 3.1 添加域名

*   路径：域名管理->添加域名

![](https://img.loadingspace.cn/blog-img/qiniu-adddomain.png)

*   如果有SSL证书，通信协议可选择https
*   源站配置默认为七牛云，选择你创建的空间bucket
*   点击保存，七牛云会自动生成一个CNAME记录值用于配置解析你设置的域名

##### 3.2 解析域名

**重点步骤！！！**

在你的服务商域名解析的地方，新增一条你设置的域名
![](https://img.loadingspace.cn/blog-img/qiniu-jiexi.png)

新增一条解析记录,记录值为七牛云给你生成的CNAME记录值
![](https://img.loadingspace.cn/blog-img/qiniu-cname.png)

##### 3.3 配置SSL证书

*   如果你已有证书，可以在路径SSL证书->上传自有证书操作
*   如果你没有证书，也可以在骑牛云购买免费证书，SSL证书->购买证书

这样选择，即可免费获取一张证书
![](https://img.loadingspace.cn/blog-img/qiniu-cert.png)

回到域名管理->配置->HTTPS配置，把证书配置上，即可支持https访问

##### 3.4 测试自定义域名访问图片

外链域名那里选择自定义的域名，随便点击一张图片的详情，复制文件链接到浏览器访问，成功显示！

现在你可以通过在你的博客网站中，使用你自己搭建的图床，自己的域名来访问图片了，再也不用担心第三方图床挂掉影响文章阅读体验了~
![](https://img.loadingspace.cn/blog-img/qiniu-pictest.png)




