# 构建高可用分布式Key-Value存储服务

## 前言

当我们构建服务端应用的时候，都会面临数据存放的问题。不同的数据类型有不同的存放方式，譬如关系型数据通常使用MySQL来存储，文档型数据则会考虑使用MongoDB，而这里，我们仅仅考虑最简单的kv（key-value）。

kv的使用场景很多，一个很典型的场景就是用户session的存放，key为用户当前的session id，而value则是用户当前会话需要保存的一些信息。因为kv的场景很多，所以选择一个好的kv服务就很重要了。

对于笔者来说，一个不错的kv服务可能仅仅需要满足如下几点就够了：

+ 协议简单
+ 高性能
+ 高可用
+ 易扩容

市面上已经有很多满足条件kv服务，但笔者秉着no zuo no die的精神，决定使用[LedisDB][1] + [xcodis][2] + [redis-failover][3]来构建一个高可用分布式kv存储服务。

## 现有解决方案

在继续说明之前，笔者想说说曾经考虑使用或者已经使用的一些解决方案。

### MySQL

好吧，别笑，我真的说的是MySQL。MySQL作为一个关系型数据库，用来存储kv性能真心一点都不差。table的结构很简单，可能如下：

```
CREATE TABLE kv (
    k VARBINARY(256),
    v BLOB,
    PRIMARY KEY(k),
) ENGINE=innodb;
```

当我还在腾讯互动娱乐部门的时候，一些游戏项目就仅仅将MySQL作为kv来使用，譬如用来存放玩家数据，游戏服务器通过玩家id读取对应的数据，修改，然后更新。鉴于腾讯游戏恐怖的用户量，MySQL能撑住直接就能说明将MySQL作为一个kv来用是可行的。

不过不知道现在还有多少游戏项目仍然采用这种做法，毕竟笔者觉得，将MySQL作为一个kv服务，有点杀鸡用牛刀的感觉，MySQL还是有点重了。

### Couchbase

[Couchbase](http://www.couchbase.com/)是一个高性能的分布式NoSQL，它甚至能支持跨data center的备份。笔者研究了很长一段时间，但最终并没有决定采用，主要笔者没信心去搞定它的代码。

### Redis

Redis是一个高性能NoSQL，它不光支持kv，同时还提供了其他的数据结构如hash，list，set，zset供外部使用。

笔者在三年前就开始使用Redis，加之Redis的代码简单，很容易就能理解掌控。所以一直到现在，笔者都会优先使用Redis来存储很多非关系型数据。自然对于kv，笔者也是采用Redis来存放的。

但Redis也有一些不足，最大的莫过于内存限制，Redis存储的总数据大小最好别超过物理内存，不然性能会有问题。同时，笔者觉得Redis的RDB和AOF机制也比较蛋疼，RDB的时候系统可能会出现卡顿，而AOF在rewrite的时候也可能出现类似的问题。

因为内存的限制，所以Redis不能存储超大量的数据，为了解决这个问题，我们只能采用cluster的方案，但是Redis官方的cluster仍然处于开发阶段，并不能真正在生产环境中使用。所以笔者开发了[LedisDB][1]。

## LedisDB

开发[LedisDB][1]，主要就是为了解决Redis内存限制问题，它主要有如下特性：

+ 采用Redis协议，大部分Redis的client都能直接使用。
+ 提供类似Redis的API，支持kv，hash，list，set，zset。
+ 底层采用多种db存储实际数据，支持rocksdb（推荐），leveldb，goleveldb，boltdb，lmdb，没有Redis内存限制问题，因为将数据放到硬盘里面了。
+ 高性能，参考[benchmark](https://github.com/siddontang/ledisdb/wiki/Benchmark)，虽然比Redis略慢，但完全可用于生产环境。

一个简单地例子：

```
//start ledis server
ledis-server 

//another shell
ledis-cli -p 6380

ledis> set a 1
OK
ledis> get a
"1"
```

可以看到，LedisDB非常类似Redis，所以用户能很方便的从Redis迁移到LedisDB上面。在实际生产环境中，笔者建议底层选择rocksdb作为其存储模块，它不光性能高，同时提供了很多配置方便用户根据特定情况进行调优（当然，理解这一堆配置可是一件很蛋疼的事情）。后续，笔者对于LedisDB的使用说明都会是基于rocksdb的。


## 数据安全

虽然LedisDB能存储大量数据，并且易于使用，但是作为一个数据存储服务，数据的安全性是一个非常需要考虑的问题。

+ LedisDB提供了dump和load工具，我们可以很方便的对其备份。在dump的时候，我们仅仅使用的是rocksdb的snapshot机制，非常快速，同时不会阻塞当前服务。这点可能是相对于Redis RDB的优势。虽然Redis的RDB在save的时候也是fork一个子进程进行处理，但如果Redis的数据量巨大，仍然可能造成Redis的卡顿。
+ LedisDB提供类似MySQL的binlog支持，任何操作都是写入binlog之后再最终提交到底层db的。如果服务崩溃，我们能通过binlog进行数据恢复。binlog文件有大小限制，当超过阀值之后，LedisDB会写入一个新的binlog中，而不是像Redis的AOF一样进行rewrite处理。
+ LedisDB支持同步或者异步replication，同步复制能保证数据的强一致，但是会牺牲系统的性能，而异步复制虽然高效，但可能会面对数据丢失问题。这其实就是一个CAP选择问题，在P（partition tolerance）铁定存在的情况下，选择C（consistency）还是选择A（availability）？通常情况下，笔者会选择A。

## 故障转移

在生产环境中，为了保证数据安全，一个master我们会通常配备一个或者多个slave（笔者喜欢将其称为replication topology），当master当掉的时候，监控系统会选择一个最优的slave（也就是拥有master数据最多的那个），将其提升为新的master，并且将其他slave指向该new master。这套流程也就是我们通常说的failover。

Redis提供了sentinel机制来实现整个replication topology的failover。但sentinel是跟redis绑定的，所以不能直接在LedisDB上面使用，所以笔者开发了[redis-failover][3]，一个能支持redis，或者LedisDB failover的sentinel。

redis-failover通过定期向master发送`role`命令来获知当前replication topology，主要是slaves的信息。当master当掉之后，redis-failover就会从先前获取的slaves里面选择一个最优的slave，提升为master，选择最优的算法很简单，通过`info`命令得到"slave_priority"和"slave_repl_offset"，如果哪个slave的priority最大，就选择那个，如果priority都一样，则选择replication offset最大的那个。

redis-failover会存在单点问题，所以redis-failover自身需要支持cluster。redis-failover的cluster在内部选举一个leader用来进行实际的monitor以及failover处理，当leader当掉之后，则进行重新选举。

现阶段，redis-failover可以通过外部的zookeeper进行leader选举，同时也支持内部自身通过raft算法进行leader选举。

## 分布式集群

我们通过使用LedisDB来解决了Redis单机数据容量问题，通过replication机制保证数据安全性，通过redis-failover用来进行failover处理，到现在为止，这套架构能很好地工作，但随着数据量的持续增大，单台机器最终无法存储所有数据了，我们不得不考虑通过cluster的方式来解决，也就是将数据放到不同的机器上面去。

要构建LedisDB的cluster，笔者考虑了如下三种方案，这里，我们不说啥hash取模或者consistency hash了，如果cluster真能通过这两种技术简单搞定，那还要这么费力干啥。

+ Redis cluster。
    
    redis cluster是redis官方提供的cluster解决方案，性能高，并且能支持resharding。可是直到现在，redis cluster仍处于开发阶段，至少笔者是不敢将其用于生产环境中。另外，笔者觉得它真的很复杂，还是别浪费脑细胞去搞定这套架构了。
    
+ 定制client。

    通过定制client，我们可以知道不同key的路由规则，自然就能找到实际的数据了。这方面的工作我的一位盆友正在进行，但定制client有一个很严重的问题在于所有的client都必须自己实现，其实不算是一个通用的解决方案。
    
+ Proxy

    记得有人说过，计算机科学领域的任何问题, 都可以通过添加一个中间层来解决。而proxy则是用来解决cluster问题的一个中间层。
    
    [Twemproxy](https://github.com/twitter/twemproxy)是一个很不错的选择，但是它不能支持resharding，而且貌似twitter内部也没在使用了，所以笔者并不考虑使用。
    
    本来笔者打算自己写一个proxy，但这时候，[codis](https://github.com/wandoulabs/codis)横空出世，它是一个分布式的proxy，同时支持resharding，并且在豌豆荚的生产环境中得到验证，笔者立刻就决定使用codis了。
    
    但codis并不支持LedisDB，同时为了满足他们自身的需求，使用的也是一个修改版的redis，鉴于此，笔者实现了[xcodis][2]，一个基于codis的，支持LedisDB以及原生redis的proxy。
    
## 最终架构

[1]: https://github.com/siddontang/ledisdb  "A Fast NoSQL"
[2]: https://github.com/siddontang/xcodis  "A distributed Redis/LedisDB proxy"
[3]: https://github.com/siddontang/redis-failover "Automatic redis monitoring and failover"
