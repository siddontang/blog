# 浮点数字节序比较

在开发LedisDB的时候，我曾考虑将zset的score使用跟redis一样的double类型，但是却没想好如何将double在底层LevelDB或者RocksDB下存储，使其能够支持zset中zrangebyscore等命令，所以只能考虑使用int64类型来代替。但在开发qdb的时候，最开始我们仍然只是支持int64，但最终通过努力，支持了double，使其能跟redis的zset api完全兼容。其实后来发现，实现很简单。

## LevelDB和ZSet

这里需要简单说明一下LevelDB（RocksDB只是它的一个衍生优化版本，但最核心基本的原理还是一样的）。

LevelDB是一个高性能的KV存储库，数据在LevelDB里面是根据key来进行排序存储的，而key和value是任意的字节数组。在外部看来，LevelDB里面的数据就是一个有序map，map里面的key是顺序存储的，如下：

```
{
	"key1" : "value1",
	"key2" : "value2",
	...
}
```

在redis里面，zset是一个有序集合，每一个member都对应一个score，member是唯一的，score则可能重复。我们可以通过score进行很多处理，譬如获取`[score1, score2]`这个区间的所有元素，或者获取某个member对应的score再整个zset里面的rank。为了实现上面的需求，我们需要在内部用一个有序的数据结构来存储score以及其对应的member，redis使用skip list来进行处理，那如何将这样的映射关系在LevelDB里面很好的处理呢？

因为LevelDB将数据是按照key顺序存储的，所以我们只需要将score的信息绑定到key上面，就能很容易的实现一个简易的skip list。对于一个zset里面的member，可能在LevelDB里面实际的key结构如下：

`key:score:member` 

对于一个zset来说，key和member都是arbitrary bytes array，可以很方便的绑定到LevelDB的key上面，但是score可是double的，如何绑定上去参与排序呢？


## LedisDB ZSet int64 score

再说double之前，先来聊聊LedisDB的做法，LedisDB使用的score是int64，对于一个int64来说，计算机内部是使用8 bytes进行存储的，下面是例子，会打印不同int64数据在小端序和大端序下8字节array十六进制表现：

```go
import "fmt"
import "math"
import "encoding/binary"

func p(a int64) {
    b1 := make([]byte, 8)
    b2 := make([]byte, 8)

    binary.LittleEndian.PutUint64(b1, uint64(a))
    binary.BigEndian.PutUint64(b2, uint64(a))

    fmt.Printf("%0x\t%0x\n", b1, b2)
}

func main() {
	p(0)
	p(1)
	p(0xFFFF0000)
	p(math.MaxInt64)
	
	p(-1)
	p(-200)
	p(-300)
	p(math.MinInt64)
}
```

输出结果如下：

```
0000000000000000    0000000000000000

0100000000000000    0000000000000001
0000ffff00000000    00000000ffff0000
ffffffffffffff7f    7fffffffffffffff

ffffffffffffffff    ffffffffffffffff
38ffffffffffffff    ffffffffffffff38
d4feffffffffffff    fffffffffffffed4
0000000000000080    8000000000000000
```

第一列是小端序打印的结果，第二列是大端序打印的结果。首先我们可以看到，对于0来说，无论什么端序，都是全0，对于正整数来说，我们可以看到，0xFFFF0000是铁定大于1的，但是在对应的小端序存储的bytes array里面，会发现如果按照字节序进行排序0100000000000000是大于0000ffff00000000的，而大端序的结果0000000000000001是小于00000000ffff0000的。对于负整数也是一样，-200是大于-300的，但是小端序的结果是小于，而大端序的结果是大于。

所以我们可以知道，对于一个整数，我们需要使用大端序的方式将其绑定到LevelDB的key上面，这样才能参与正常的排序。但是我们又发现，如果直接这样处理，负数的大端序结果是铁定大于正数的，譬如-1的ffffffffffffffff就大于1的0000000000000001，所以为了正常的处理整数的排序，我们只需要在key上面加一个前缀标志就可以了。LedisDB里面做了如下处理：

```
key<38ffffffffffffff:member
key<ffffffffffffffff:member
key=0000000000000000:member
key=0000000000000001:member
```

因为`=`的ascii值大于`<`，我们将正整数和0前面加上`=`，将负数前面加上`<`，这样在进行字节序排序的时候，正数和0一定能排在负数的后面，也就是完全能满足从小到大排序的需求了。

## QDB ZSet double score

上面可以知道，我们使用大端序加上一个前缀标志，就能很好的处理int64类型的score的排序，但double可没有这么简单，不然LedisDB早就支持了。

首先我们来看看double类型IEEE754规范，对于一个double来说，使用8 bytes存储，格式如下：

+ 符号: 1 bit （0为正，1为负）
+ 指数: 11 bits
+ 分数: 52 bits


![double format](http://upload.wikimedia.org/wikipedia/commons/thumb/a/a9/IEEE_754_Double_Floating_Point_Format.svg/1236px-IEEE_754_Double_Floating_Point_Format.svg.png)

对于一个64 bits的double，如果指数为e，那么它对应的值计算方式为:

![](http://upload.wikimedia.org/math/9/7/c/97c7aee4ad33a9763c30f49628648779.png)

如果指数越大，那么double的值越大，如果分数越大，在相同指数的情况下面double值也越大。所以我们如果将double使用大端序存放到8字节array里面，我们就能直接进行字节序比较，但这个仅限于正的double情况下。因为对于负的double，它除了最高位bit为1这一点不一样之外，其余完全跟相反的正数底层存储方式一模一样，所以如果我们直接将其转为大端序比较，会发现结果是相反的。一个简单地例子:

```go
func d(a float64) {
    b := make([]byte, 8)

    binary.BigEndian.PutUint64(b, math.Float64bits(a))

    fmt.Printf("%064b\n", binary.BigEndian.Uint64(b))
}

func main() {
	d(0)
	
	d(1)
	d(2)
	
	d(-1)
	d(-2)
}
```

结果如下：

```
0000000000000000000000000000000000000000000000000000000000000000
0011111111110000000000000000000000000000000000000000000000000000
0100000000000000000000000000000000000000000000000000000000000000
1011111111110000000000000000000000000000000000000000000000000000
1100000000000000000000000000000000000000000000000000000000000000
```

这里在插入一片广告文章，推荐阅读，[这里](http://www.cygnus-software.com/papers/comparingfloats/comparingfloats.htm)。

当初LedisDB就是因为没有办法很好的处理负double的排序比较问题，才采用了int64，但是真的没有办法吗？为啥int64类型的负数能通过大端序进行字节序比较呢？我们知道，在计算机内部，负数是使用补码存储的，也就是反码加1，负数的反码，就是在源码的基础上，符号位不变，其余各个位取反。对于-1来说，它的源码为[10000001]，反码为[11111110]，补码为[11111111]。（为了简单，使用int8表示的）。就因为它有了取反的这一步，我们才能直接用大端序字节序比较。

所以对于double的负数，我们也仅仅需要干一件事情，就是取反，那么它的大端序字节序比较就是正常的了。在QDB里面，对于一个double，我们做了如下处理:

```go
// We can not use lexicographically bytes comparison for negative and positive float directly.
// so here we will do a trick below.
func float64ToUint64(f float64) uint64 {
	u := math.Float64bits(f)
	if f >= 0 {
		u |= 0x8000000000000000
	} else {
		u = ^u
	}
	return u
}

func uint64ToFloat64(u uint64) float64 {
	if u&0x8000000000000000 > 0 {
		u &= ^uint64(0x8000000000000000)
	} else {
		u = ^u
	}
	return math.Float64frombits(u)
}
```

如果一个double数据值大于等于0，那么我们将最高位bit设置为1，而对于负数，我们直接取反，就这一步简单的操作，就能让我们完美的支持double的score了。

## 总结

让QDB的zset支持double score，其实很简单的一件事情，但我在开发LedisDB的时候竟然没有想到如何去解决。我们很多时候，往往追求高大上的东西，譬如架构，设计模式等，但忽略了计算机最基本的底层知识（上面这些其实也就是大小端序，补码，反码等），所以有时候还真的重新好好把最基本的东西给回顾一下。

最后，在推荐一下QDB，QDB是一个类似Redis的KV数据库，底层使用LevelDB/RocksDB作为存储引擎，解决了Redis内存限制问题，同时能支持string，hash，list，set和zset这几种数据结构，性能卓越，你也可以认为它是LedisDB的升级版本。

网址 [https://github.com/reborndb/qdb](https://github.com/reborndb/qdb)。