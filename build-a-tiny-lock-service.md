# 构建一个简易的中心化锁服务

## 为什么需要锁服务？

有时候，在分布式系统中，不同的服务实例需要操作同一份资源，所以我们需要一套机制保证对该资源并发操作的数据一致性。

最通常的做法，就是lock。各个服务在操作资源之前，首先lock住该资源，处理完成之后在释放lock。也就是说，我们通过lock使得并行操作串行化，保证了资源的数据一致性。

## 为什么要实现重新实现一个？

虽然现在有很多不错的锁服务实现，譬如基于Zookeeper，Etcd，甚至Redis，但这里我仍然想自己实现一个简易的lock服务，主要有以下几点原因：

### Multi Lock

同时获取多个lock，有时候我们想同时对多个资源进行操作，所以需要多把lock，但是无论是Zookeeper还是Redis，我们都需要对其进行多次调用，影响整体性能。

### Hierarchical Lock

支持hierarchical lock，我称之为path lock，我们的系统是有类似文件夹的概念的，假设现在我们需要往文件夹a/b/c下面增加一个文件，我们就必须得保证另一个进程不会同时操作该文件夹以及其祖先和后继（譬如删除a，或者a/b，或者也往a/b/c下面增加文件），但是允许操作其兄弟（譬如在a/b/d下面增加文件）。

仍然是上面那个例子，假设我们需要操作a/b/c，首先我们需要对a加read lock，然后对a/b加read lock，最后在对a/b/c加write lock，如果这个通过Zookeeper来实现，真心很麻烦，而且性能存疑。

## TLock

tlock是一个简单的中心化lock service，现阶段为了性能没有High Availability支持，所以如果当掉了后果还是有点严重的。:-)

tlock支持两种模式，key和path，key就是最通常的对某个资源进行操作，而path则是我上面说的Hierarchical Lock。同时tlock支持Multi Lock。

tlock提供Restful API以及RESP(Redis Serialization Protocol) API，所以你可以通过任意HTTP客户端或者Redis客户端使用。

一个简单的HTTP例子：

```
// shell1

// 同时lock a，b，c三个资源，如果30s之后仍没lock成功，返回超时错误
// 返回lockid供后续unlock使用
POST http://localhost/lock?names=a,b,c&type=key&timeout=30

// Unlock
DELETE http://localhost/lock?id=lockid

// shell2
POST http://localhost/lock?names=a,b,c&type=key&timeout=30

DELETE http://localhost/lock?id=lockid
```

一个简单的RESP例子：

```
# shell1 redis-cli
redis>LOCK abc TYPE key TIMEOUT 10
redis>lockid
redis>UNLOCK lockid
redis>OK

# shell2 redis-cli 
redis>LOCK abc TYPE key TIMEOUT 10
// 一直挂起直到shell1 unlock 
redis>lockid
redis>UNLOCK lockid
redis>OK
```

tlock同时提供了RESP的客户端：

```
import "github.com/siddontang/tlock"

client := NewRESPClient(addr)
locker, _ := client.GetLocker("key", "abc")
locker.Lock()
locker.Unlock()
```

## Todo

可以看到，tlock是一个非常简单的服务，虽然是一个单点并且没有HA支持，但是已经能满足我们项目的需求，毕竟我们只是需要一个简单的锁服务。

后续，我可能会基于Zookeeper尝试实现path lock，虽然这样HA能够保证，但是性能怎样，到时候压测了再说吧。