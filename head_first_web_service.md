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

一个简单的例子：KV Storage Service，后面就叫KV吧。类似于S3，只是我们不是存文件，而是元数据。后面我们就用KV来表明服务的名字吧。

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

既然是Web service，自然选用HTTP来做交互，比起自己实现一套不通用的协议，或者使用Google Protocol Buffers这些的，HTTP具有太多的优势，虽然它的性能稍微有点差，数据量稍微有点臃肿，但几乎所有的浏览器以及数不清的库能直接支持，想想都有点小激动了。

所以我们唯一要做的，就是设计好我们的API，让外面更方便的使用Web service。

### API

根据wiki的定义，Web service通常有两种架构方式，RESTful和Arbitrary（RPC，SOAP，etc）。

REST是Representational state transfer的缩写，而满足REST架构模型的我们通常称之为Restful：

+ 使用URI来表示资源，譬如http://example.com/user/1代表ID为1的user。
+ 使用标准HTTP方法GET，POST，PUT，DELETE等来操作资源，譬如Get http://example.com/user/1 来获取user 1的信息，而使用Delete http://example.com/user/1 来删除user 1。
+ 支持资源的多种表现形式，譬如上例Get中设置Content-Type为json，让服务端返回json格式的user信息。

相对于Restful，另一种则是Arbitrary的，我不熟悉SOAP，这里就以RPC为例。

RPC就是remote procedure call，它通过在HTTP请求中显示的制定需要调用的过程名字以及参数来与服务端进行交互。仍然是上面的例子，如果我们需要得到用户的信息，可能就是这样Get http://example.com/getuser?userid=1，如果要删除一个用户，没准是这样Get http://example.com/delUser?userid=1。

那选择何种架构呢？在这里，我倾向使用Restful架构模型，很大原因在于它理解起来很容易，而且实现简单，而现在越来越多的Web service提供的API采用的是Restful模式，从另一个方面也印证了它的流行。

所以这个Web service的接口就是这样了：

    GET http://kv.com/key
    DELETE http://kv.com/key
    POST http://kv.com/key -dvalue
    PUT http://kv.com/key -dvalue
    
上面POST和PUT可以等价，如果key存在，则用value覆盖，不存在则新建。

### 架构

好了，扯了这么多，我们也要开始搭建我们的Web service了。因为采用的是HTTP协议，所以我们可以直接使用现成的HTTP server来帮我们处理HTTP请求。譬如nginx，apache，不过用go或者python直接写一个也不是特别困难的事情。

我们还需要一个storage server用来存放key-value，mysql可以，redis也行，或者我的[ledisdb](http://ledisdb.com)，谁叫红薯说可以打广告的。

最开始，我们就只有一台机器，启动一个nginx用来处理HTTP请求，然后启动一个ledisdb用来存放数据。然后开始对外happy的提供服务了。

KV开始工作的很好，突然有一天，我们发现随着用户量的增大，一台机器处理不过来了。好吧，我们在加一台机器，将nginx和ledisdb放到不同的机器上面。

可是好景不长，用户量越来越多，压力越来越大，我们需要再加机器了，因为nginx是一个无状态的服务，所以我们很容易的将其扩展到多台机器上面去运行，最外层通过DNS或者LVS来做负载均衡。但是对于有状态的服务，譬如上面的ledisdb，可不能这么简单的处理了。好吧，我们终于要开始扯到分布式了。

### CAP

在聊分布式之前，我们需要知道CAP定理，因为在设计分布式系统的时候，CAP都是必须得面对的。

+ Consistency，一致性
+ Avaliability，可用性
+ Partition tolerance，分区容忍性

CAP的核心就在于在分布式系统中，你不可能同时满足CAP，而只能满足其中两项，但在分布式中，P是铁定存在的，所以我们设计系统的时候就需要在C和A之间权衡。

譬如，对于MySQL,它最初设计的时候就没考虑分区P，所以很好的满足CA，所以做过MySQL proxy方面工作的童鞋应该都清楚，要MySQL支持分布式是多么的蛋疼。

而对于一般的NoSQL，则是倾向于采用AP，但并不是说不管C，只是允许短时间的数据不一致，但能达到最终一致。

而对于需要强一致的系统，则会考虑牺牲A来满足CP，譬如很多系统必须写多份才算成功，

### Replication

对于前面提到的Ledisdb，因为涉及到数据存放，本着不要把鸡蛋放到一个篮子里面的原则，我们也不能将数据放到一台机器上面，不然当机了就happy了。而解决这个的办法就是replication。

熟悉MySQL或者Redis的童鞋对replication应该都不会陌生，它们的replication都采用的是异步的方式，也就是在一段时间内不满足数据一致性，但铁定能达到最终一致性。

但如果真想支持同步的replication，怎么办呢？谁叫我们容不得数据半点丢失。通常有几种做法：

+ 2PC，3PC
+ Paxos，Raft

因为这方面的坑很深，就不在累述，不过我是很推崇Raft的，相比于2PC，3PC，以及Paxos，Raft足够简单，并且很好理解。有机会在说明吧。

### 水平扩展

好了，通过replication解决了ledisdb数据安全问题，但总有一天，一台机器顶不住了，我们要考虑将ledisdb的数据进行拆分到多台机器。通常做法如下：

+ 最简单的做法，hash(key) % num，num是机器的数量，但这种做法在添加或者删除机器的时候会造成rehash，导致大量的数据迁移。
+ 一致性hash，它相对于传统的hash，在添加或者删除节点的时候，它能尽可能的少的进行数据迁移。不过终归还是有数据流动的。
+ 路由映射表，不同于一致性hash，我们在外部自己负责维护一张路由表，这样添加删除节点的时候只需要更改路由表就可以了，相对于一致性hash，个人感觉更加可控。

我个人比较喜欢预分配+路由表的方式来进行水平扩展，所谓预分配，就是首先我就将数据切分到n个（譬如1024）shard，开始这些shard可以在一个node里面，随着node的增加，我们只需要迁移相关的shard，同时更新路由表就可以了。这种方式个人感觉灵活性最好，但对程序员要求较高，需要写出能自动处理resharding的健壮代码。

好了，解决了replication，解决了水平扩展，很长一段时间我们都能happy，当然坑还是挺多的，遇到了慢慢再填吧。

## 没有提到的关键点

+ Cache，无论怎样，cache在服务器领域都是一个非常关键的东西，用好了cache，你的服务能处理更多的并发访问。facebook这篇paper [Scaling Memcache at Facebook](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf)专门讲解了相关知识，那是绝对的干货。
+ 消息队列，当并发量大了之后，光靠同步的API调用已经满足不了整个系统的性能需求，这时候就该MQ上场了，譬如RabbitMQ都是不错的选择。
+ 很多很多其他的。。。。。。。这里就不列举了。

## 总结

上面只是我对于Web service一点浅显的见解，如果里面的知识稍微对你有用，那我已经感到非常高兴了。但就像实践才是检验真理的唯一标准一样，理论知道的再多，还不如先弄一个Web service来的实在，反正现在国内阿里云，腾讯云，百度云啥的都不缺，缺的只是跑在上面的好应用。