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

Redis是一个高性能NoSQL，它不光支持kv，同时还提供了其他的数据结构如hash，set，zset，list供外部使用。

笔者在三年前就开始使用Redis，加之Redis的代码简单，很容易就能理解掌控。所以一直到现在，笔者都会优先使用Redis来存储很多非关系型数据。自然对于kv，笔者也是采用Redis来存放的。

但Redis也有一些不足，最大的莫过于内存限制，Redis存储的总数据大小最好别超过物理内存，不然性能会有问题。同时，笔者觉得Redis的RDB和AOF机制也比较蛋疼，RDB的时候系统可能会出现卡顿，而AOF在rewrite的时候也可能出现类似的问题。

因为内存的限制，所以Redis不能存储超大量的数据，为了解决这个问题，我们只能采用cluster的方案，但是Redis官方的cluster仍然处于开发阶段，并不能真正在生产环境中使用。所以笔者开发了[LedisDB][1]。

## LedisDB

## 数据安全

## 故障转移

## 分布式集群

## 最终架构

[1]: https://github.com/siddontang/ledisdb  "A Fast NoSQL"
[2]: https://github.com/siddontang/xcodis  "A distributed Redis/LedisDB proxy"
[3]: https://github.com/siddontang/redis-failover "Automatic redis monitoring and failover"
