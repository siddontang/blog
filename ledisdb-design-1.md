[ledisdb](https://github.com/siddontang/ledisdb)是一个用go实现的基于leveldb的高性能nosql数据库，它提供多种数据结构的支持，网络交互协议参考redis，你可以很方便的将其作为redis的替代品，用来存储大于内存容量的数据（当然你的硬盘得足够大！）。

同时ledisdb也提供了丰富的api，你可以在你的go项目中方便嵌入，作为你app的主要数据存储方案。

## 与redis的区别

ledisdb提供了类似redis的几种数据结构，包括kv，hash，list以及zset，（set因为我们用的太少现在不予支持，后续可以考虑加入），但是因为其基于leveldb，考虑到操作硬盘的时间消耗铁定大于内存，所以在一些接口上面会跟redis不同。

最大的不同在于ledisdb对于在redis里面可以操作不同数据类型的命令，譬如（del，expire），是只支持kv操作的。也就是说，对于del命令，ledisdb只支持删除kv，如果你需要删除一个hash，你得使用ledisdb额外提供的hclear命令。

为什么要这么设计，主要是性能考量。leveldb是一个高效的kv数据库，只支持kv操作，所以为了模拟redis中高级的数据结构，我们需要在存储kv数据的时候在key前面加入相关数据结构flag。

譬如对于kv结构的key来说，我们按照如下方式生成leveldb的key：

    func (db *DB) encodeKVKey(key []byte) []byte {
        ek := make([]byte, len(key)+2)
        ek[0] = db.index
        ek[1] = kvType
        copy(ek[2:], key)
        return ek
    }
    
kvType就是kv的flag，至于第一个字节的index，后面我们在讨论。   

如果我们需要支持del删除任意类型，可能的一个做法就是在另一个地方存储该key对应的实际类型，然后del的时候根据查出来的类型再去做相应处理。这不光损失了效率，也提高了复杂度。

另外，在使用ledisdb的时候还需要明确知道，它只是提供了一些类似redis接口，并不是redis，如果想用redis的全部功能，这个就有点无能为力了。

## db select

redis支持select的操作，你可以根据你的业务选择不同的db进行数据的存放。本来ledisdb只打算支持一个db，但是经过再三考虑，我们决定也实现select的功能。

因为在实际场景中，我们不可能使用太多的db，所以select db的index默认范围就是[0-15]，也就是我们最多只支持16个db。redis默认也是16个，但是你可以配置更多。不过我们觉得16个完全够用了，到现在为止，我们的业务也仅仅使用了3个db。

要实现多个db，我们开始定了两种方案：

- 一个db使用一个leveldb，也就是最多ledisdb将打开16个leveldb实例。
- 只使用一个leveldb，每个key的第一个字节用来标示该db的索引。

这两种方案我们也不知道如何取舍，最后决定采用使用同一个leveldb的方式。可能我们觉得一个leveldb可以更好的进行优化处理吧。

所以我们任何leveldb key的生成第一个字节都是存放的该db的index信息。

## KV

kv是最常用的数据结构，因为leveldb本来就是一个kv数据库，所以对于kv类型我们可以很简单的处理。额外的工作就是生成leveldb对应的key，也就是前面提到的encodeKVKey的实现。


## Hash

hash可以算是一种两级kv，首先通过key找到一个hash对象，然后再通过field找到或者设置相应的值。

在ledisdb里面，我们需要将key跟field关联成一个key，用来存放或者获取对应的值，也就是key:field这种格式。
 
这样我们就将两级的kv获取转换成了一次kv操作。

另外，对于hash来说，（后面的list以及zset也一样），我们需要快速的知道它的size，所以我们需要在leveldb里面用另一个key来实时的记录该hash的size。
    
hash还必须提供keys，values等遍历操作，因为leveldb里面的key默认是按照内存字节升序进行排列的，所以我们只需要找到该hash在leveldb里面的最小key以及最大key，就可以轻松的遍历出来。

在前面我们看到，我们采用的是key:field的方式来存入leveldb的，那么对于该hash来说，它的最小key就是**"key:"**，而最大key则是**"key;"**，所以该hash的field一定在**"(key:, key;)"**这个区间范围。至于为什么是**“;”**，因为它比**":"**大1。所以**"key:field"**一定小于**"key;"**。后续zset的遍历也采用的是该种方式，就不在说明了。

## List

list只支持从两端push，pop数据，而不支持中间的insert，这样主要是为了简单。我们使用key:sequence的方式来存放list实际的值。

sequence是一个int整形，相关常量定义如下：

    listMinSeq     int32 = 1000
    listMaxSeq     int32 = 1<<31 - 1000
    listInitialSeq int32 = listMinSeq + (listMaxSeq-listMinSeq)/2

也就是说，一个list最多存放1<<31 - 2000条数据，至于为啥是1000，我说随便定得你信不？

对于一个list来说，我们会记录head seq以及tail seq，用来获取当前list开头和结尾的数据。

当第一次push一个list的时候，我们将head seq以及tail seq都设置为listInitialSeq。

当lpush一个value的时候，我们会获取当前的head seq，然后将其减1，新得到的head seq存放对应的value。而对于rpush，则是tail seq + 1。

当lpop的时候，我们会获取当前的head seq，然后将其加1，同时删除以前head seq对应的值。而对于rpop，则是tail seq - 1。

我们在list里面一个meta key来存放该list对应的head seq，tail seq以及size信息。

## ZSet

zset可以算是最为复杂的，我们需要使用三套key来实现。

- 需要用一个key来存储zset的size
- 需要用一个key:member来存储对应的score
- 需要用一个key:score:member来实现按照score的排序

这里重点说一下score，在redis里面，score是一个double类型的，但是我们决定在ledisdb里面只使用int64类型，原因一是double还是有浮点精度问题，在不同机器上面可能会有误差（没准是我想多了），另一个则是我不确定double的8字节memcmp是不是也跟实际比较结果一样（没准也是我想多了），其实更可能的原因在于我们觉得int64就够用了，实际上我们项目也只使用了int的score。

因为score是int64的，我们需要将其转成大端序存储，这样通过memcmp比较才会有正确的结果。同时int64有正负的区别，负数最高位为1，所以如果只是单纯的进行binary比较，那么负数一定比整数大，这个我们通过在构建key的时候负数前面加"<"，而正数（包括0）加"="来解决。所以我们score这套key的格式就是这样：

    key<score:member //<0
    key=score:member //>=0
    
对于zset的range处理，其实就是确定某一个区间之后通过leveldb iterator进行遍历获取，这里我们需要明确知道的事情是leveldb的iterator正向遍历的速度和逆向遍历的速度完全不在一个数量级上面，正向遍历快太多了，所以最好别去使用zset里面带有rev前缀的函数。

## 总结

总的来说，用leveldb来实现redis那些高级的数据结构还算是比较简单的，同时根据我们的压力测试，发现性能还能接受，除了zset的rev相关函数，其余的都能够跟redis保持在同一个数量级上面，具体可以参考ledisdb里面的性能测试报告以及运行ledis-benchmark自己测试。

后续ledisdb还会持续进行性能优化，同时提供expire以及replication功能的支持，预计6月份我们就会实现。

ledisdb的代码在这里[https://github.com/siddontang/ledisdb](https://github.com/siddontang/ledisdb)，希望感兴趣的童鞋多多反馈。