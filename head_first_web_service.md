应[红薯](http://my.oschina.net/javayou)的邀约，决定给某大学的童鞋讲讲Web Service相关知识，鉴于是第一次在学校献丑，所以还是老老实实的准备，先把类似逐字稿的东西写出来，然后在准备PPT吧。

关于Web service，这个话题太广太泛，加之我也只熟悉一些特定的领域，所以准备从两方面入手，1，什么是Web service，就当是概念性的介绍，让大家有个相关认识。2，则是根据一个简单的例子，告诉大家如何构建一个Web service服务。

## 什么是Web service

首先根据Wiki的定义：**A Web Service is a method of communication between two electronic devices over a network.** 

简单来说，Web Service就是基于网络不同设备之间互相通信的一种方式。Web Service是一个软件服务，它提供很多API，而客户端通过Web协议进行调用从而完成相关的功能。

Web service并不是一个新奇的概念，相反从很早的分布式计算，到网格计算，到现在的云，都或多或少的有着Web service的影子。只不过随着近几年一浪高过一浪的互联网热潮以及Google，Amazon等公司的大力推动，Web service变得愈发流行。

越来越多的公司开始提供Web service，而同时又有更多的公司基于这些Web service提供了更加上层的Web service。

Amazon的S3（Simple Storage Service）是一个文件存储服务，用户通过S3将文件存放到Amazon的服务器上面，Amazon负责保证该文件的安全（包括不被别人获取，不丢失等）。而Drew Houston则在S3的基础上，构造了一个令人惊奇的同步网盘：Dropbox，同时，Dropbox又将相关API提供出去，供其他的Application其同步服务。

可以看到，正是因为有了越来越多的Web services，才让我们现在的互联网生活变得越发精彩。

## 实现一个简单的Web service

好了，上面扯了这么多，是不是心痒痒想自己开发一个Web service？开发一个Web service并不是那么容易的事情，尤其是涉及到分布式之后。不过我觉得一个小例子没准就能说明很多东西。当然我自认并不是Web service的专家（这年头专家架构师太多，我只能算打酱油的），很多东西难免疏漏，并且一些设计也会带有很强烈的个人色彩，如果大家有啥更好的认识，欢迎跟我讨论（妹子优先！）。

一个简单的例子：KV Storage Service。类似于S3，只是我们不是存文件，而是元数据。后面我们就用KV来表明服务的名字吧。

对于KV来说，它只会涉及到三种操作，如果用代码表示如下：

```
//根据指定的key获取对应的value
Get(key)

//设置key的值为value，如果key本来存在，则更新，否则新建
Put(key, value)

//删除key
Delete(key)
```

### 交互协议

### API

### 原始架构

## 分布式Web service

### 无状态 and 有状态

### CAP

### 水平扩展

### Replication

## 没有提到的关键点

+ Cache，无论怎样，cache在服务器领域都是一个非常关键的东西，用好了cache，你的服务能处理更多的并发访问。facebook这篇paper [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)专门讲解了相关知识，那是绝对的干货。
+ 消息队列，当并发量大了之后，光靠同步的API调用已经满足不了整个系统的性能需求，这时候就该MQ上场了，譬如RabbitMQ都是不错的选择。
+ NoSQL，这玩意现在也火的一塌糊涂，但它并不是SQL的终结者，而是另一种解决方案。这里推荐下我的[LedisDB](http://ledisdb.com)，谁叫红薯说最后可以打广告的。
+ 很多很多其他的。。。。。。。这里就不列举了。

## 总结

上面只是我对于Web service一点浅显的见解，如果里面的知识稍微对你有用，那我已经感到非常高兴了。但就像实践才是检验真理的唯一标准一样，理论知道的再多，还不如先弄一个Web service来的实在，反正现在国内阿里云，腾讯云，百度云啥的都不缺，缺的只是跑在上面的好应用。