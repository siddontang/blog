# SmartStack: Airbnb的自动服务发现和注册框架

## Micro-service引发的问题

在几年前做企业快盘的时候，为了构建一个高可用的分布式系统，我们采用了一个模块一个服务，不同服务之间通过HTTP交互的架构模型，除了数据存储服务（MySQL，Redis等），我们其他的服务都是无状态的，这样就能非常方便的进行水平扩展，满足不断增加的业务需求。现在才知道，这种架构模式就是俗称的micro-service，如果当初我们就能拿这些概念出去忽悠，没准能让自己的产品更加高大上一点的。:-)

micro-service虽然方便，毕竟各个模块是相互独立，我们可以独立开发，独立部署，只要约定好相互之间的HTTP Restful API就成了。但是，随着服务的增多，我们会面临一个问题，就是某一个服务到底在哪里？我们如何才能发现该服务并进行调用。

在项目初期，服务数量不多的情况下面，我们可以将所有服务的地址写到一个配置文件里面（或者更改hosts），部署升级的时候通过puppet或者其它工具进行整个系统的更新，但这样，就会面临一个问题，任何增加删除服务的操作，都可能引起整个系统的更新。随着系统规模的扩大，服务数量的增多，谁都知道不可行了。

Airbnb的工程师一定也碰到了类似了问题，否则他们不会开发了SmartStack。关于SmartStack的详细介绍，可以参考这篇文章[SmartStack: Service Discovery in the Cloud](http://nerds.airbnb.com/smartstack-service-discovery-cloud/)，国内的oschina已经有相关翻译[SmartStack 介绍 —— 云端的服务发现](http://www.oschina.net/translate/smartstack-service-discovery-cloud)，所以我就不用考虑再将这篇文章重复翻译一遍了。下面我只是想说说我的一些理解。

## 不怎么好的解决方案

Airbnb的这篇文章同时列出了一些不怎么好的解决方案，个人觉得对我们设计分布式系统也是很有借鉴意义的。

### DNS

DNS应该是一个非常简单地解决方案了，一个服务配置一个Domain，通过DNS解析，客户端对某个服务的请求就能被路由到某一台具体的机器上面处理。

虽然很简单，但是DNS也有很严重的问题，首先就是延时，如果我们更新了DNS，我们无法保证一些客户端能及时的收到DNS变更的消息。同时，在机器上面，DNS通常都有缓存，所以更加增大了变更DNS的延时。

另外，DNS是随机路由的，我们不能自定义自己的路由算法，所以很有可能我们会面临一个服务里面，一些机器繁忙的在处理请求，而另一些机器则几乎被闲置。

还有一个很严重的问题，一些应用程序譬如Nginx在程序启动的时候会缓存DNS的结果，也就是如果不做任何特殊处理，DNS的任何变更对当前运行中的Nginx是完全无效的。

综上，如果服务的节点动态变更比较频繁，使用DNS来进行服务发现并不是一个很好的解决方案。但对于一些节点长时间不会变动的服务，譬如Zookeeper，使用DNS则是一个比较好的方式。

### 中心化的负载均衡

我们可以使用一个中心化的负载均衡器来进行服务的路由。所有的路由信息都存储到这个负载均衡器里面。但我们如何发现这个负载均衡器，通常的做法就是DNS，不过这又会出现上面DNS的问题。

同时，因为这个负载均衡器是一个中心化的节点，必然面临单点性能问题，而且如果负载均衡器当掉了，我们就会面临整个系统不可用的问题了（这时候完全不知道其他服务在哪里了）。

因为负载均衡器是一个单点，所以我们需要考虑将其做HA处理，另外，我们还需要考虑性能问题。如果有钱，考虑购入F5，不过这种硬件路由方案真心很贵。LVS, Nginx或者HAProxy也是一个不错的选择。如果在AWS上面，可以考虑使用ELB，但只能针对外网IP。

**其实中心化的负载均衡方案，后续可以演化为无状态proxy方案，我们通过zookeeper或者其他coordinator来存储整个集群的路由信息，并通过2PC的方式同步更新到所有proxy上面，因为proxy是无状态，所以非常方便的进行水平扩展。当然proxy的发现也是一个问题，我们可以使用DNS，也可以使用LVS。**

### 服务自己内部注册并发现

其实依赖zookeeper等协调服务来处理的。服务会自己注册到zookeeper上面，其他的服务节点通过zookeeper的watch方式实时的感知到整个集群的变化。

当然，使用这种方式也是有限制的。如果我们使用同一种语言进行开发，譬如java，那么每个服务嵌入一个zookeeper的client library是很容易的，但如果用了不同的语言，譬如有go，node.js，等等，让所有服务都能很好的跟zookeeper进行交互就比较困难了。

同时，如果我们希望一些其他第三方应用（譬如Nginx）也能享受到zookeeper的好处，这就比较麻烦了，因为这些服务压根不支持zookeeper。

另外，即使zookeeper存储了整个路由信息，我们仍然没有很好的办法定制路由算法，即使连最简单地round-robin都没法很好的支持，譬如一个客户端将第一个请求发给了第一个节点，如何将第二个请求发给第二个节点？

总之，单纯地使用zookeeper是不可能的，但正如我在前面说的，**我们可以引入一个无状态的proxy，由proxy负责跟zookeeper打交道。**

## SmartStack: Nerve and Synapse

Airbnb列出了通用的三种不可取的做法，我想他们应该是都尝试过并且踩了坑，所以才有了后续的SmartStack。:-）

SmartStack由两个模块构成，nerve和synapse，当然还依赖zookeeper和haproxy。

部署的时候，service跟nerve一起部署，client则跟synapse以及local haproxy一起部署。

nerve负责管控其对应的service，并通过ephemeral的方式挂载到zookeeper上面，这样如果nerve所在的机器当掉，或者nerve负责的service挂掉了（nerve会每隔一段时间进行service存活检测），nerve与zookeeper的连接就会断开，外部就能感知节点的动态变更了。

synapse负责watch zookeeper的变更，当获取到节点变更事件之后，将最新的路由信息更新到本机local haproxy上面，客户端要访问对应的service，都是通过local haproxy路由到相应节点进行访问。

可以看到，SmartStack是一个非常简单地实现方式，但是它很好的解决了分布式系统中服务的发现与注册问题。SmartStack通过nerve进行服务的注册以及注销，通过synapse + local haproxy的方式进行服务的发现。如果一个client要访问对应的服务，只能通过local haproxy，这里local haproxy有点类似于中心化的负载均衡器，但是它仅仅限于本机，所以不存在DNS以及单点等问题。

## SmartStack的不足

SmartStack的实现是很巧妙的，并且非常的简单，但是它也有一些自身的问题，最主要的就是友好的可用性问题。这套系统依赖了zookeeper，需要额外部署nerve，synapse以及local haproxy，运维上面略显复杂，相比较而言，Consul（Hashicorp的分布式服务发现程序）就只有一个二进制文件，部署更加方便。（后续如果有时间，可以写写SmartStack vs Consul）

为什么我特别关注运维，主要在于现在我们开发的一个分布式kv：RebornDB也存在同样的问题，运维步骤略显繁琐，导致很多时候我们开发自己都烦如何很好的构建一个可运行环境。

不过，这年头，因为有了docker以及mesos，kubernate等技术，没准运维的复杂度会降低很多。

## 总结

总的来说，SmartStack是一个很好的服务发现解决方案，原理非常简单，但功能却很强大。不过，如果我们将local haproxy往上提升变成一个haproxy cluster，这不就跟我前面说的无状态proxy差不多呢？没准后续也可以用go整一个出来，毕竟SmartStack的ruby代码看得我挺头大的。