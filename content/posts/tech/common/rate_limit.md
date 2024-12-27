---
title: "被限流了？如何避免触发大模型流式对话API的QPS限制？"
date: 2024-12-27T15:29:00+08:00
lastmod: 2024-12-27T15:29:00+08:00
author: ["Max"]
keywords: 
- AI
- LLM
- 大墨西哥
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
    image: "" #图片路径例如：posts/tech/123/123.png
    zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
    caption: "" #图片底部描述
    alt: ""
    relative: false
---

### 分享技术，用心生活
--- 

# 背景

国内的公有大模型想要调用，基本都会有QPS的限制。那么如何实现我们在请求时候，有针对性的进行限流？被限流的请求又如何知道它的状态？是在队列中排队还是已经被消费在生成内容？本文将通过Redis的lua、zset+guava的RateLimiter来实现。

# 目的

*   针对大模型的QPS限制，在用户发起请求时，在业务侧可以满足QPS的要求

*   在超出QPS时，剩余请求进入队列排队，且方便页面展示请求状态（排队中或正在生成中）

# 实现原理

*   当请求大模型API时

1.  请求进入队列，并在redis中记录一个k-v；key为**业务唯一id**，value为当前队列的请求排队数。
2.  新增当前业务请求的计数器zset：**业务唯一id**作为zset的**value**和初始化**score=0**（score业务上的含义为：从请求的时间点起，已经消费请求的个数。）

*   用户侧展示请求状态时

1.  获取第一步骤中的k-v值，记为`rateLimitPointCount`
2.  获取第二步骤中的score值，记为`rateLimitReduceCount`
3.  比较`rateLimitPointCount`和`rateLimitReduceCount`大小，即可判断出请求此时处于已消费还是排队中状态

*   通过redis的zset来存储每个业务请求**排队点**

*   通过redis的incr实现消费**自增**

*   通过guava的RateLimit实现令牌桶中**令牌**的管理

# 实现过程

### 定义大模型限流工具类

```java
@Component
public class LLMRateLimiter {
    // 最大QPS
    private static final int MAX_QPS = 5;
    // 请求队列最大长度
    private static final int MAX_QUEUE_SIZE = 1000;
    private static final RateLimiter rateLimiter = RateLimiter.create(MAX_QPS);
    private static final ExecutorService executorService = Executors.newFixedThreadPool(2);
    public static LinkedBlockingQueue<Runnable> requestQueue = new LinkedBlockingQueue<>(MAX_QUEUE_SIZE);
```

设置qps=5，并初始化guava的**RateLimit**，`RateLimiter.create(5)`表示创建一个令牌容量为5的令牌桶，且每秒新增5个令牌；初始化线程池用于消费；初始化请求队列，设置最大长度。

### 请求放入队列（LLMRateLimiter工具类）

```java
public static Boolean addRequest(LLMChatParam llmChatParam, Consumer<LLMChatParam> doChat) throws InterruptedException {
    // 队列已满，新请求丢弃
    if (requestQueue.size() >= MAX_QUEUE_SIZE) {
        return false;
    }
    // 请求放入队列
    requestQueue.put(() -> doChat.accept(llmChatParam));
    // 初始化zset中元素-业务唯一id
    StringRedisUtils.add("rate_counter", llmChatParam.getRateLimitReduceKey(), 0);
    // 消费队列
    processTasks();
    return true;
}

```

### 标记当前队列大小（Redis工具类）

```java
public static void rateLimit(String key) {
    int size = LLMRateLimiter.requestQueue.size();
    // 大模型队列大小标记点key
    String rateLimitKey = SystemConstants.RedisKeyEnum.RATE_LIMIT_POINT.getKey(key);
    // 记录当前大模型队列大小
    set(rateLimitKey, Integer.toString(size));
}
```

以上为在请求入口处，使用Redis工具类使用`rateLimit`标记当前队列大小
同时使用`LLMRateLimiter.addRequest`将请求放入队列。

### 消费（LLMRateLimiter工具类）

```java
private static void processTasks() {
    executorService.submit(() -> {
boolean acquire = rateLimiter.tryAcquire(5, 1, TimeUnit.SECONDS);
if (acquire) {
    Runnable task = requestQueue.poll();
    if (task != null) {
        task.run();
    }
    // 队列消费后，所有计数器加1
    StringRedisUtils.incrementAllScoresByLua("rate_counter");
}
    });
}
```

通过自定义的线程池来异步消费请求，guava的`rateLimiter.tryAcquire`在1s内获取5个令牌。

### 更新所有计数器（Redis工具类）

所谓的计数器，即redis中维护的`zset`,1个请求=1个计数器，更新计数器即更新zset中元素的`score`值。有3种实现方式

1.  ~~循环zset实现：性能较差，不建议使用~~
2.  Lua脚本实现：能够保证原子性，且效率最高
3.  redis的pipeline实现：减少命令次数，效率较高

第一种我们直接弃用，不采纳。那么可能有读者就疑问了，既然lua脚本效果最好，为什么不直接用呢,还要用pipeline。如果你使用阿里云Redis的话，那么这个坑我先帮你踩了。
首先笔者第一方案确实是用lua实现的，且在测试环境（自建redis服务）执行也是ok的；但是，上线后大量用户反馈所有的请求都是**排队中**状态，经过排查发现，用户的请求其实已经处理完毕，大模型也返回了响应的内容。继续排查发现所有的计数器score值都为0，也就是初始化后一直没有成功更新。查看日志后，发现lua脚本在生产环境执行失败了。
提工单给阿里云，反馈如下

![工单](https://img.loadingspace.cn/blog-img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241126101929.png)

查看官方文档说明

![官方文档](https://img.loadingspace.cn/blog-img/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241126101907.png)
最终真相大白！既然有限制，又没有时间去研究文档中如何正确使用lua;所以最快的修复方式就是采用另一种方式去实现。下面把2种实现方式都展示出来，方便大家自取。

*   lua脚本实现

```java
public static void incrementAllScoresByLua(String key) {
    RedisScript<Long> script = new DefaultRedisScript<>(
            "local key = KEYS[1]\n" +
                    "\n" +
                    "local elements = redis.call('ZRANGE', key, 0, -1, 'WITHSCORES')\n" +
                    "for i = 1, #elements,2 do\n" +
                    "    local member = elements[i]\n" +
                    "    redis.call('ZINCRBY', key, 1, member)\n" +
                    "end\n");

    STRING_REDIS_TEMPLATE.execute(script, Collections.singletonList(key));

}
```

*   redis的pipeline实现

```java
    public static void incrementAllScoresByLua(String key) {
        Set<String> members = STRING_REDIS_TEMPLATE.opsForZSet().range(key, 0, -1);
        STRING_REDIS_TEMPLATE.executePipelined(new SessionCallback<Object>() {
            @Override
            public Object execute(RedisOperations operations) throws DataAccessException {
                ZSetOperations zSetOperations = operations.opsForZSet();
                for (String member : members) {
                    zSetOperations.incrementScore(key, member, 1.0);
                }
                return null;
            }
        });
    }
```

两种方式原理都是通过获取zset中所有元素->循环,`incr`命令计数器加1。

### 请求状态返回（Redis工具类）

```java
public static Map<String, Object> rateLimitStatus(String key) {
    Map<String, Object> map = new HashMap<>(2);
    String rateLimitPointKey = SystemConstants.RedisKeyEnum.RATE_LIMIT_POINT.getKey(key);
    String rateLimitNeedReduceCountKey = SystemConstants.RedisKeyEnum.RATE_LIMIT_NEED_REDUCE_COUNT.getKey(key);
    Double rateLimitReduceCount = score("rate_counter", rateLimitNeedReduceCountKey) == null ? 0d : score("rate_counter", rateLimitNeedReduceCountKey);
    Double rateLimitPointCount = StringUtils.isEmpty(get(rateLimitPointKey)) ? 0d : Double.parseDouble(get(rateLimitPointKey));
    // 代表此次生成之前的队列已经消费完毕，开始消费本次生成的队列
    if (Double.compare(rateLimitReduceCount, rateLimitPointCount) >= 0) {
        map.put("rateLimit", Constants.RateLimitEnum.END.getType());
        // 删除计数器
        StringRedisUtils.removeZset("rate_counter", rateLimitNeedReduceCountKey);
        // 删除此次记录点
        StringRedisUtils.delete(rateLimitPointKey);
    } else {
        map.put("rateLimit", Constants.RateLimitEnum.ING.getType());
    }
    return map;
}
```

在页面中需要呈现状态时，使用此方法。通过比较标记点队列大小和计数器大小，可得出请求状态。同时在消费后，删除对应的计数器和标记点。

### 效果展示

*   排队中状态

![排队中](https://img.loadingspace.cn/blog-img/queue1.gif)

*   消费中状态

![消费中](https://img.loadingspace.cn/blog-img/queue2.gif)

### 总结

本文使用了`queue`和`RateLimiter`实现限流、`zset`和`lua脚本`实现请求状态计算。但同时以上也存在着对代码侵入性较大。需要在每个请求处都要添加一个标记点，在每个查询状态处都要添加查询代码。读者可以自行通过**自定义注解和AOP**等其他方式实现，减少对代码的入侵。如果大家有更好的方案，欢迎在留言区讨论。





