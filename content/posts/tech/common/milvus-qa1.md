---
title: "milvus实践-快问快答"
date: 2024-05-10T22:03:44+08:00
lastmod: 2024-05-10T22:03:44+08:00
author: ["Max"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- mulvus
- 向量数据库
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
---

>前言：主要问答收集于社区交流群中的对话问答

---

Q：milvus standalone部署，客户端新增collection时，报错超时, 查看milvus日志，发现大量报 rocksmq consume too slowly是什么原因？

A：集合/分区创建太多，消息系统的压力会很大，用partition key来替换多分区方案；或者是磁盘性能不好，更换支持NVMe的固态盘

Q：不同milvus集群之间或单机迁移到集群怎样进行数据迁移？

A：使用milvus backup工具：https://github.com/zilliztech/milvus-backup
（原理：milvus-backup是用api从milvus server拿到collection schema，然后再拿到segments list，然后从minio上把那些segment一个个拷出来，然后上传到另一个minio bucket里）；可参考用户方案：https://www.yuque.com/zhi-docs/dev/qgbwafhonynfdgy5?singleDoc=#%20%E3%80%8AMilvus%20%E8%B7%A8%E9%9B%86%E7%BE%A4%E6%95%B0%E6%8D%AE%E8%BF%81%E7%A7%BB%E3%80%8B

Q：单独进行query可以查到想要的结果，单独进行search也能查到想要的结果，但是混合查询就不出想要的结果了（用的用的ivf_sq8索引）

A：ivf索引的混合查询是会这样的。ivf索引就像是把一所小学的小学生分班管理，nlist就是班级数量，假设我们按年龄大小分成nlist个班级，你查询时的nprobe是要查找的班级。你要找的人大约有12岁，所以就从高年级里选出nprobe个班，然后你又加了个查询条件“我要找名字叫李明的小朋友”。
可惜，那nprobe个班级里没有叫“李明”的小盆友，那就查不到了。
如果你把nprobe设成nlist的值，相当于全校搜索，那就能查出来了

Q：请教一下，https://zhuanlan.zhihu.com/p/473617910 里提到“存储过程是以 Segment 为单位，用的是列存的方式，每个 Primary Key 、Column、Vector 都是单独用一个文件存储”
这里的列式具体是什么形式的呢？是ORC、Parquet还是其他什么格式呢？

A：类似于MySQL里的binlog

Q：目前我们使用的的机器是8c16g,目前就4个collection，每个collection就2个partition，数据量最多也就40w。但这里实际上会涉及频繁的删除、插入。但我实际上用col.query( expr="", output_fields = ["count(*)"],)查询数据量并没有增长很多，但是memory 一直涨。

A：删除的时候，是标记删除，只有完成compaction之后，数据才会被真正清理掉；可以隔几天调用一次compaction接口。

Q：milvus不会自动compaction，是么？

A：会自动compaction，不过需要达到触发条件。比如一个segment里面有多少比例的数据被删除，才会做compaction

Q：删除的数据还能被查到是为什么？

A：查询的时候设置consistency_level="Strong"

Q：不同的一致性对性能差距很大吗？

A：strong一般比eventually或者bounded多耗费几百毫秒

Q：问一下这个kafka connector是什么意思，是一个独立组件，可以把kafka里的数据导入milvus吗，具体的文档有吗？

A：博客：https://zilliz.com/blog/announce-confluent-kafka-connector-for-Milvus-and-Zilliz-unlock-power-of-real-time-ai

网页：https://zilliz.com/product/integrations/confluent

 github 代码：https://github.com/zilliztech/kafka-connect-milvus

Q：upsert功能，是根据主键id来确定要更新的行是哪一行吗？可以指定别的标量字段作为更新的判断条件吗？

A：只能根据主键

Q：milvus产生的数据有几种？

A：有两种，一种是minio中的数据文件，另一种是etcd中的元数据，这两者在运行时是保持一致的；元数据记录了哪些表哪些分片的数据保存在minio中的路径

Q：请问，GPU版的 milvus 使用docker-compose V2 部署完成以后，进行压测时，发现仍然使用的产CPU，这有可能会是什么原因？

A：带GPU_前缀的才能用gpu搜索，例如：GPU_IVF_FLAT

Q：milvus如何保证高可用？

A：milvus支持，k8s分布式部署，多副本，backup，cdc，基于这些功能和组件可以去实现高可用

Q：怎么让collection常驻内存吗？不常驻内存的话是不是每个用户在前端查询的时候都要load一次，计算资源消耗很大啊

A：调用collection.load()就会让表数据常驻内存，只调用一次即可，它会自动加载新增数据。对已经loaded的表再次调用collection.load()不会有什么效果

Q：load一次后，多个用户并发计算相似度的时候，内存中还是只加载了一个collection吗？

A：默认条件下，一个表只会在内存里有一份，除非你在调用load()时设置了replicas参数大于1

Q：没有数据输入的情况下仍有1-3g的波动是什么原因？

A：数据停止输入之后，有一段时间内存波动很正常，后台有复杂的各种任务，比如compaction和建索引。假设你连续插入10GB数据，停止输入数据，很可能后续10分钟里都在干着各种后台任务

Q：想问下 filter中 比如我要增加一个搜索条件  text like '%电动车%'

我发现 目前一个bug就是只能在 text like '电动车%'  这样情况下搜索出来东西
text like '%电动车%' 和 text like '%电动车' 都搜索不出来 是什么原因啊？

A：2.3版本的字符串索引只支持前缀匹配。2.4才会支持其他的匹配方式。

Q：milvus最多支持多大数据量的索引？

A：milvus数据索引具体支持的数据量取决于硬件资源和部署方式，节点资源占用可以通过 sizing tool https://milvus.io/tools/sizing/ 进行计算，通常情况下 8G 内存可以支持超过 5m 的 128dim 向量数据和 1m 的 768dim 数据。性能优化参见：https://mp.weixin.qq.com/s/4gDsAF4QnmXWzomrSFRLLg

Q：化学式检索系统有没有完整的安装教程？

A：可参考教程：https://github.com/towhee-io/examples/tree/main/medical/molecular_search

Q：count(＊)统计的真实的数据量吗？

A：count(*)是精确值，但它得出的结果会受consistency_level的影响。如果Strong，则是完全精确的。如果是Bounded/Eventually，有可能少数数据还在pulsar里未消费完成，就不被计入行数

Q：主键重复的数据算在里面吗？

A：算的（有规划要做主键去重）

Q：在做rag时，经常将在milvus搜索到的结果再进行一次rerank操作，实现把最相关的内容排在最前面或者最后面，让大模型能够抓住最重要内容。但是milvus搜索到的结果本身就可以根据向量值进行排序，为何还需要rerank的操作呢？

A：如果是单路召回，主要是embedding模型的精排，因为embedding粗排阶段是双塔模型，精度差一些。reranker一般是单塔模型（例如：bge-reranker），效果好、计算量大。多路召回的话就是来重排序的；简单的做 mixing 的话可以用 milvus 自带的算法 https://milvus.io/docs/reranking.md

Q：如何理解多路召回的话就是来重排序的？

A：多路召回有n个排序，可以根据权重来设置每一路的得分来融合这n个排序。但是比较通用的方法就是把这n个排序召回的文段再用rerank模型精排一下

Q：storageType: local # please adjust in embedded Milvus: local, available values are [local, remote, opendal], value minio is deprecated, use remote instead
这个配置是否指milvus可以不用minio，直接存储在本机磁盘？

A：使用embedded milvus，这里才配置为 local。如果想用本地磁盘来作为milvus的存储介质，可以参考官网这里的教程：https://milvus.io/docs/install_standalone-docker.md

Q：为什么IP索引distances大于1？

A：需要对向量做归一化

Q：RAG准确率能从那些方面优化？

A：向量模型质量、向量维度、索引类型设置、检索召回配置、多路召回、改写查询、reranking、finetune等，可参考：https://zilliz.com/learn/how-to-enhance-the-performance-of-your-rag-pipeline

Q：RAG准确率能从那些方面优化？

A：向量模型质量、向量维度、索引类型设置、检索召回配置、多路召回、改写查询、reranking、finetune等

Q：milvus已支持动态schema，请问可以基于动态字段建立index吗？

A：动态字段不能是vector

Q：请问一下，如何在已经创建的集合中再新增一个字段？

A：集合一旦创建，schema是没法改的，增减字段都是不允许的。不过如果你打开了dynamic schema，集合里面会有一个特殊的JSON类型字段可用于存额外的key-value，效果上就有点像动态的增加字段，可参考：https://milvus.io/docs/enable-dynamic-field.md 

Q：请问一下，在所有搜索条件都不变的情况下，topk选择分别是10条和20条，返回的结果分别为5条和10条是怎么一回事呢？

A：如果加了过滤条件的话，返回的条数有可能小于topk，比如设置了查询分区条件




