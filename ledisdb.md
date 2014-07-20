应vash的要求，我决定写一篇LedisDB的文章，方便让他了解LedisDB，以便于更好的参与开发。

## 介绍

LedisDB是一个高性能的NoSQL，使用GoLang进行开发，API参考Redis设计，底层采用LevelDB进行持久化存储。

## 特性

LedisDB具有很多不错的特性，完全可以作为替代Redis的另一种选择方案。

+ 丰富的数据结构支持：KV，List，Hash，ZSet，Bitmap。
+ 采用LevelDB作为底层存储，突破内存限制。
+ 数据Expire以及TTL支持，过期自动删除。
+ Redis（redis-cli）客户端支持。
+ 多个Client API支持，现包括Golang，Python以及Lua。
+ 可作为Package嵌入到Golang应用中使用。
+ Replication支持，设置Slave更好的保障数据。
+ 不断丰富的外部工具，譬如dump，binlog解析等。

## 起因

我们项目中大量的使用了Redis，但是当数据量越来越大，Redis内存不足问题开始慢慢显现出来，虽然可以通过集群方案来解决，但我们仍然希望单机能承载更多数据。

另外，虽然Redis存储了大量的数据，但是在大部分时候，整个应用只需要少量的热点数据，这点尤其体现在存储用户相关信息上面，我们用Redis存放了三千多万的用户数据，但即使在一天的高峰期，同时在线也就几十万人，剩余的两千多万的数据内存白白浪费了。

最开始我们想到的是使用MySQL存储数据，而Redis只是作为缓存使用，但这样就会面临MySQL和Redis数据更新的问题，如果数据首先存放到MySQL，如何自动更新到Redis，或者首先存放到Redis，又如何更新到MySQL？

所以，我们真正需要的是一个Redis的替代品，它能够将数据存放到硬盘，同时将热点数据放置在内存供外部快速使用，如果能提供Redis的数据结构那就更加强大了。

幸运的是，已经有很多相关的实现了，我们也调研了两个:

+ [SSDB](https://github.com/ideawu/ssdb)
+ [Redis-Storage](https://github.com/qiye/redis-storage)

它们都是很优秀的项目，但当我们深入之后，突然觉得，为什么我们不也用Golang自己开发一个？于是就有了LedisDB。

## 安装

+ 安装Golang，推荐使用Golang 1.3的最新版本，性能提升很大，GC也做了很多优化。

+ 建立工具目录并Check LedisDB的源码：

        mkdir $WORKSPACE
        cd $WORKSPACE
        git clone git@github.com:siddontang/ledisdb.git src/github.com/siddontang/ledisdb

        cd src/github.com/siddontang/ledisdb

+ 安装LevelDB以及Snappy，在LedisDB的源码里面提供了一个简易的安装脚本， ```sh build_leveldb.sh```。如果你自己安装LevelDB，需要注意，我调整了一些LevelDB的代码，用以提升性能，可做参考：

        + db/dbformat.h
        
        // static const int kL0_SlowdownWritesTrigger = 8;
        static const int kL0_SlowdownWritesTrigger = 16;
        
        // static const int kL0_StopWritesTrigger = 12;
        static const int kL0_StopWritesTrigger = 64;
        
        + db/version_set.cc
        
        //static const int kTargetFileSize = 2 * 1048576;
        static const int kTargetFileSize = 32 * 1048576;
        
        //static const int64_t kMaxGrandParentOverlapBytes = 10 * kTargetFileSize;
        static const int64_t kMaxGrandParentOverlapBytes = 20 * kTargetFileSize;

+ 在dev.sh里面设置正确的LevelDB以及Snappy安装路径，执行```source dev.sh```。

+ 安装LedisDB依赖的外部库，执行```source bootstap.sh```

+ 构建LedisDB，执行```go install ./...```

## 使用

LedisDB可以作为单独的NoSQL服务使用，同时也可以作为一个嵌入式Package集成到你的应用中使用。

LedisDB从一开始就是设计成Redis友好的，服务协议采用的Redis协议，而接口也是尽量的保持跟Redis一致，熟悉Redis的开发人员会很容易的上手使用。

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

