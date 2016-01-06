## 起因

[ledisdb](https://github.com/siddontang/ledisdb)是一个参考ssdb，采用go实现，底层基于leveldb，类似redis的高性能nosql数据库，提供了kv，list，hash以及zset数据结构的支持。

我们现在的应用极大的依赖redis，但随着我们用户量越来越大，redis的内存越来越不够用，并且replication可能还会导致超时问题。虽然后续我们可以通过添加多台机器来解决，但是在现有机器配置下面，我们仍希望单台机器承载更多的用户。另外，因为业务的特性，我们其实并不需要将所有的数据放到内存，只需要存放当前活跃用户。

经过我们的调研，发现ssdb已经很好的帮我们解决了这个问题，它提供了跟redis一致的接口（当然有些地方还是稍微不同），但是底层采用leveldb进行存储。根据其官网的描述，性能已经接近甚至超越了redis。

本着[造轮子](http://blog.csdn.net/siddontang/article/details/24765201)的精神，我决定用go实现一个类似的db，取名为ledisdb，也就是**level-redis-db**，为啥不用现成的ssdb，我觉得有如下几个原因：

+ go语言开发的快速，这点毋庸置疑，虽然性能上面铁定离c++的代码有差距，但是我能够快速的进行原型搭建并实验。实际上，我在很短的时间里面就开发出了ledisdb，让我后续继续开发有了信心。
+ leveldb的研究，我一直很想将leveldb应用到我们的项目中，作为本机热点数据的首选数据存储方式，通过ledisdb，让我对leveldb的使用有了很多经验。
+ redis的熟悉，虽然我用了很久的redis，但是很多redis的命令仍然需要去查手册，通过实现ledisdb，我更加熟悉了redis的命令，同时，因为要了解这个命令redis如何实现，对redis内部又重新来了一次剖析。

在准备开发ledisdb的时候，我就在思索一个问题，我需不需要开发另一个redis？其实这是一个很明确的问题，我不需要另一个redis。ledisdb虽然参考了redis，但为了实现简单，有时候我做了很多减法或者变更，譬如对于zset这种数据结构，我就只支持int64类型的score，而redis的score是double类型的，具体原因后续讲解zset的时候详细说明。

所以，我们可以认为，ledisdb是一个基于redis通信协议，提供了多种高级数据结构的nosql数据库，它并不是另一个redis。

## 编译安装

因为ledisdb是用go写的，所以首先需要安装go以及配置GOROOT，GOPATH。

    mkdir $WORKSPACE
    cd $WORKSPACE
    git clone git@github.com:siddontang/ledisdb.git src/github.com/siddontang/ledisdb

    cd src/github.com/siddontang/ledisdb

    #构建leveldb，如果已经安装了，可忽略
    ./build_leveldb.sh  
    
    #安装ledisdb go依赖
    . ./bootstap.sh     
    
    #配置GOPATH等环境变量
    . ./dev.sh          
    
    go install ./... 
    
具体的安装说明，可以查看代码目录下面的readme。

## Example

使用ledisdb很简单，只需要运行：

    ./ledis-server -config=/etc/ledis.json
   
ledisdb的配置文件采用json格式，为啥选用json，我在[使用json作为主要的配置格式](http://blog.csdn.net/siddontang/article/details/23595817)里面有过说明。

我们可以使用任何redis客户端连接ledisdb，譬如redis-cli，如下：

    127.0.0.1:6380> set a 1
    OK
    127.0.0.1:6380> get a
    "1"
    127.0.0.1:6380> incr a
    (integer) 2
    127.0.0.1:6380> mset b 2 c 3
    OK
    127.0.0.1:6380> mget a b c
    1) "2"
    2) "2"
    3) "3"

## leveldb

因为leveldb是c++写的，所以在go里面需要使用，cgo是一种很好的方式。这里，我直接使用了[levigo](https://github.com/jmhodges/levigo)这个库，并在上面进行了封装，详见[这里](http://blog.csdn.net/siddontang/article/details/24359873)。虽然有一个go-leveldb，无奈仍不能用。

cgo的性能开销还是有的，这点在我做benchmark的时候就明显感觉出来，不过后续优化的空间很大，譬如将多个leveldb的调用逻辑该用c重写，这样只需要一次cgo就可以了。不过这个后续在考虑。

leveldb的一些参数在构建编译的时候是需要调整的，这点我没啥经验，只能google和参考ssdb。譬如下面这几个：

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

相关参数的调优，只能等我后续深入研究leveldb了在好好考虑。


## 性能测试

任何一个服务端服务没有性能测试报告那就是耍流氓，我现在只是简单的用了redis_benchmark进行测试，测试环境为一台快两年的老爷mac air机器。

测试语句：

    redis-benchmark -n 10000 -t set,incr,get,lpush,lpop,lrange,mset -q
    
redis-benchmark默认没有hash以及zset的测试，后续我在自己加入。

leveldb配置：
    
    compression       = false
    block_size        = 32KB
    write_buffer_size = 64MB
    cache_size        = 500MB

### redis

    SET: 42735.04 requests per second
    GET: 45871.56 requests per second
    INCR: 45248.87 requests per second
    LPUSH: 45045.04 requests per second
    LPOP: 43103.45 requests per second
    LPUSH (needed to benchmark LRANGE): 44843.05 requests per second
    LRANGE_100 (first 100 elements): 14727.54 requests per second
    LRANGE_300 (first 300 elements): 6915.63 requests per second
    LRANGE_500 (first 450 elements): 5042.86 requests per second
    LRANGE_600 (first 600 elements): 3960.40 requests per second
    MSET (10 keys): 33003.30 requests per second

### ssdb

    SET: 35971.22 requests per second
    GET: 47393.37 requests per second
    INCR: 36630.04 requests per second
    LPUSH: 37174.72 requests per second
    LPOP: 38167.94 requests per second
    LPUSH (needed to benchmark LRANGE): 37593.98 requests per second
    LRANGE_100 (first 100 elements): 905.55 requests per second
    LRANGE_300 (first 300 elements): 327.78 requests per second
    LRANGE_500 (first 450 elements): 222.36 requests per second
    LRANGE_600 (first 600 elements): 165.30 requests per second
    MSET (10 keys): 33112.59 requests per second

### ledisdb

    SET: 38759.69 requests per second
    GET: 40160.64 requests per second
    INCR: 36101.08 requests per second
    LPUSH: 33003.30 requests per second
    LPOP: 27624.31 requests per second
    LPUSH (needed to benchmark LRANGE): 32894.74 requests per second
    LRANGE_100 (first 100 elements): 7352.94 requests per second
    LRANGE_300 (first 300 elements): 2867.79 requests per second
    LRANGE_500 (first 450 elements): 1778.41 requests per second
    LRANGE_600 (first 600 elements): 1590.33 requests per second
    MSET (10 keys): 21881.84 requests per second
    
可以看到，ledisdb的性能赶redis以及ssdb还是有差距的，但也不至于不可用，有些差别并不大。至于为啥lrange比ssdb高，我比较困惑。

后续的测试报告，我会不断在benchmark文件里面更新。

## Todo。。。。。。

[ledisdb](https://github.com/siddontang/ledisdb)还是一个非常新的项目，比起ssdb已经在生产环境中用了很久，还有很多路要走，还有一些重要的功能需要实现，譬如replication等。

欢迎有兴趣的童鞋一起参与进来，在漫漫程序开发路上有一些好基友可是很幸运的。