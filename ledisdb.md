应vash的要求，我决定写一篇LedisDB的文章，方便让他了解LedisDB，以便于更好的参与开发。

## 介绍

LedisDB是一个高性能的NoSQL，使用GoLang进行开发，API参考Redis设计，熟悉Redis的程序员会非常容易上手使用。

## 特性

LedisDB具有很多不错的特性，完全可以作为替代Redis的另一种选择方案。

+ 丰富的数据结构：KV，List，Hash，ZSet，Bitmap。
+ 可选择多种数据库（LevelDB，goleveldb，LMDB，RocksDB）作为底层存储，突破内存限制。
+ 数据Expire以及TTL支持。
+ Redis（redis-cli）客户端支持。
+ 多个Client API支持，现包括Golang，Python以及Lua。
+ 可作为Package嵌入到Golang应用中使用。
+ Replication支持。

## 起因

我们项目中大量的使用了Redis，但是当数据量越来越大，Redis内存不足问题开始慢慢显现出来，虽然可以通过集群方案来解决，但我们仍然希望单机能承载更多数据。

另外，虽然Redis存储了大量的数据，但是在大部分时候，整个应用只需要少量的热点数据，这点尤其体现在存储用户相关信息上面，我们用Redis存放了三千多万的用户数据，但即使在一天的高峰期，同时在线也就几十万人，剩余的两千多万的数据内存白白浪费了。

所以，我们真正需要的是一个Redis的替代品，它将数据存放到硬盘，同时缓存热点数据，并且支持Redis丰富的数据结构，还有更重要的一点，能够嵌入到Go程序里面作为Package使用。

幸运的是，业界已经有很多产品供我们参考了，我们调研了以下两个:

+ [SSDB](https://github.com/ideawu/ssdb)
+ [Redis-Storage](https://github.com/qiye/redis-storage)

它们都是很优秀的项目，但是上面我说了，我们最需要的是一个可嵌入Go程序的Package，所以就有了LedisDB。

## 安装和使用

具体安装使用，请大家直接查看[ReadMe](https://github.com/siddontang/ledisdb/blob/master/README.md)，这里不再详述。


作为服务的例子：

    ./ledis-server -config=/etc/ledis.json

    //another shell
    ledis-cli -p 6380
    
    ledis 127.0.0.1:6380> set a 1
    OK
    ledis 127.0.0.1:6380> get a
    "1"
 
**启动ledis-server之后，你甚至可以通过redis的客户端redis-cli直接使用。**

作为Package的例子：

    import "github.com/siddontang/ledisdb/ledis"
    l, _ := ledis.Open(cfg)
    
    //select a db with index
    db, _ := l.Select(0)

    db.Set(key, value)

    db.Get(key)

