# 分布式系统的时间

## 事件的顺序

大家都知道，Linearizability在一些系统（譬如分布式数据库）里面是非常重要的，我们不能允许数据更新之后仍然能读到原先的值，譬如银行转账，用户A有100元，转给用户B 10元，这个操作之后用户A只可能有90元了，但如果A后续发起了另一个转账请求给C转10元的时候，事务里面读到A的余额仍然是100元，这个问题就大了。

一个更简化的说明，假设在某个时间点t我们写入了一条数据，譬如`set a = 10`，那么在t之后，所有的read a读到的都应该是10，而不是a以前的值。

那么，我们如何确定read a读到的一定是最新的数据呢？在一个系统里面，如果一个事件a发生在另一个事件b的前面，我们就可以认为a happended before b, 可以用 a -> b来表示。
    
如果a和b是在同一个进程里面执行，（注：这里我们假定进程是顺序执行event的），那么我们可以很容易的就知道a在b之前发生，因为a在进程里面是先执行的。
    
但在分布式系统里面，a和b可能在不同的进程里面（当然这两个进程一定会相互通讯），这件事情就不是特别容易判断了，我们得找到一个基准用来衡量到底谁先谁后。

我们首先就想到的是时间，我们只要知道a和b发生的时间就能够知道先后顺序了，但是我们都知道，每台机器的时间都不是一致的，虽然能通过NTP协议进行相互校对，但总有误差。所以我们不能直接使用系统时间来判断事件的先后顺序。
    
## Logic Clock

Lamport（这个大牛就不用多介绍了）在上世纪70年代，就提出了使用[Logic Clock](http://www.ics.uci.edu/~cs230/reading/time.pdf)来确定事件的先后顺序。

我们可以认为Logic Clock就是一个不断增长的number，假设两个进程P1和P2，两个事件a和b，我们用C(a)，C(b)来表示这两个事件的logic clock。

如果a和b都是在P1里面发生，如果a -> b，那么我们一定可以知道 C(a) < C(b)。

现在复杂的是a在P1里面发生，但b在P2里面发生，首先我们需要明确P1和P2一定能够相互通讯，a发生之后，P1会给P2发送相关消息，同时会带上C(a)，P2接受到这个消息之后再处理b，因为P2明确知道这个消息是从P1发过来的，如果P2当前的clock为C(old)并且小于或者等于C(a)，那么会将自己的clock更新为大于C(a)的任意值，不过通常就是C(a) + 1了，这时候在执行b，我们就一定能知道C(b) > C(a)了。

当然，上面a和b必须是相关的事件，也就是a -> b，如果a和b是独立的，那么P1和P2就不需要进行相关的交互了。

可以看到，Logic Vector的原理是非常简单的，但是它因为没有实际的物理时间概念，所以如果我们想根据某一个真实的时间来查询相关事件，这个办不到了。

在Logic Clock之后，人们又引入了[Vector Clock](http://zoo.cs.yale.edu/classes/cs426/2012/lab/bib/fidge88timestamps.pdf)，但vector clock也有logic clock同样的问题，不能依据真实的时间来查询，这里就不多介绍vector clock了。

## True Time

前面我们说了，NTP是有误差的，而且NTP还可能出现时间回退的情况，所以我们不能直接依赖NTP来确定一个事件发生的时间。在[Google Spanner](http://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)里面，通过引入True Time来解决了分布式时间问题。

Spanner通过使用GPS + Atomic Clock来对集群的机器进行校时，精度误差范围能控制在ms级别，通过提供一套TrueTime API给外面使用。

TrueTime API很简单，只有三个函数:

| Method | Return |
| ------ | ------ |
| TT.now() | TTinterval: [earliest, latest] |
| TT.after(t) | true if t has definitely passed |
| TT.before(t) | true if t has definitely not arrived |

首先now得到当前的一个时间区间，spanner不能得到精确的一个时间点，只能得到一段区间，但这个区间误差范围很小，也就是ms级别，我们用ε来表示，也就是[t - ε, t + ε]这个范围，

假设事件a发生绝对时间为tt.a，那么我们只能知道tt.a.earliest <= tt.a <= tt.a.latest, 所以对于另一个事件b，只要tt.b.earliest > tt.a.latest，我们就能确定b一定是在a之前发生的，也就是说，我们需要等待大概2ε的事件才能去提交b，这个就是spanner里面说的commit wait time。

可以看到，虽然spanner引入了TrueTime可以得到全球范围的时序一致性，但相关事务在提交的时候会有一个wait时间，只是这个时间很短，而且spanner后续都准备将其优化到 ε < 1ms，也就是对于关联事务，仅仅在上一个事务commit之后等待2ms之后就能执行，性能还是很强悍的。

但spanner有一个最大的问题，TrueTime是基于硬件的，而现在对于很多企业来说，是没有办法搞定这套部署的。所以如果Google能将TrueTime的硬件设计开源，那我觉得更加造福社区了。

## Hybrid Logic Clock

既然TrueTime这种硬件方案很多人搞不定，那么我们就采用软件方案了。

[Cockroachdb](https://github.com/cockroachdb/cockroach)使用了[Hybrid Logic Clock（HLC）](https://www.cse.buffalo.edu/tech-reports/2014-04.pdf)来解决分布式时间的问题。

HLC是基于NTP的，但它只会读取当前系统时间，而不会去修改，同时HLC又能保证在NTP出现同步问题的时候仍能够很好的进行容错处理。对于一个HLC的时间t来时，它总是大于等于当前的系统时间，并且与其在一个很小的误差范围里面，也就是 |l - pt| < ε。

HLC由两部分组成，physical clock + logic clock，l.j维护的是节点j当前已知的最大的物理时间，c.j则是当前的逻辑时间。那么判断两个事件的先后顺序就很容易了，先判断物理时间pt，在判断逻辑时间ct。

HLC的算法如下，在节点j上面：

+ 初始化: `l.j = 0, c.j = 0`
+ 给另一个进程发送或者处理自己的事件：

    ```c
    l'.j = l.j;
    // 跟当前系统时间比较，得到pt
    l.j = max(l'j, pt.j)
    // 如果pt没有变化，则c.j加1，如果有变化，因为这时候
    // 铁定PT变大了，所以我们可以将ct清零
    if (l.j = l'.j) {
        c.j = c.j + 1
    } else {
        c.j = 0
    }
    
    // Timestamp with l.j, c.j
    ```
+ 接受某一个节点m的消息事件

    ```c
    l'.j = l.j;
    // 跟当前系统事件以及节点m的pt比较，得到pt
    l.j = max(l'.j, l.m, pt.j)
    if (l.j = l'.j = l.m) {
        // pt一样，获取最大的ct，并加1
        c.j = max(c.j, c.m) + 1
    } else if (l.j = l'j) {
        // 这里表明j原来的pt比m大，只需要增加ct
       c.j = c.j + 1
    } else if (l.j = l.m) {
        // 这里表明m的pt比j原来的要大，所以直接可以用m的ct + 1
        c.j = c.m + 1
    } else {
        // pt变化了，ct清零
        c.j = 0
    }
    
    // Timestamp with l.j, c.j
    ```
    
具体的实现算法，可以看cockroachdb的[HLC实现](https://github.com/cockroachdb/cockroach/blob/master/util/hlc/hlc.go)。

HLC虽然方便，它毕竟是基于NTP的，所以如果NTP出现了问题，可能导致HLC与当前系统pt的时间误差过大，其实已经不怎么精确了，HLC论文提到对于一些out of bounds的message可以直接忽略，然后加个log让人工后续处理，而cockroachdb是直接打印了一个warning log。

## Timestamp Oracle

无论上面的Ture Time还是Hybrid Logic Time，都是为了在分布式情况下获取全局唯一时间，如果我们整个系统不复杂，而且没有spanner那种跨全球的需求，有时候一台中心授时服务没准就可以了。

在Google [Percolator](http://static.googleusercontent.com/media/research.google.com/en//pubs/archive/36726.pdf)系统这，他们就提到使用了一个timestamp oracle（TSO）的服务来提供统一的授时服务，为啥叫oracle，我猜想可能底层用的就是oracle数据库。。。

使用TSO的好处在于因为只有一个中心授时，所以我们一定能确定所有时间的时间，但TSO需要关注几个问题：

+ 网络延时，因为所有的事件都需要从TSO获取时间，所以TSO的场景通常都是小集群，不能是那种全球级别的数据库。
+ 性能，TSO是一个非常高频的操作，但鉴于它只干一件事情，就是授时，通常一个TSO每秒都能支持10w+以上的QPS，而这个对很多应用来说是绰绰有余的。
+ 容错，TSO是一个单点，所以不得不考虑容错，而这个现在基于zookeeper，etcd也不是特别困难的事情。

所以，如果我们没法实现TrueTime，同时又觉得HLC太复杂，但又想获取全局时间，TSO没准是一个很好的选择，因为它足够简单高效。

我们现在的数据库产品就使用的是TSO方案，但也不排除以后为了支持全球同步，而考虑使用TureTime或者HLC的方案。

## 最后

在分布式领域，我们很自然的会想到用时间来解决事件的时序问题，但如何保证时间的获取是全局一致并且正确的，并不是一件容易的事情，笔者认为，只要能搞定时间问题，后面编写Linearizability的系统就比较容易了。当然，如果我们还需要支持分布式事务，还有更多的事情需要考虑的，如果后面有时间，在慢慢总结吧。



