对于使用SQL或者NoSQL的童鞋来说，replication都是一个避不开的话题，通过replication，能极大地保证你的数据安全性。毕竟谁都知道，不要把鸡蛋放在一个篮子里，同理，也不要把数据放到一台机器上面，不然机器当机了你就happy了。

在分布式环境下，对于任何数据存储系统，实现一套好的replication机制是很困难的，毕竟[CAP](http://en.wikipedia.org/wiki/CAP_theorem)的限制摆在那里，我们不可能实现出一套完美的replication机制，只能根据自己系统的实际情况来设计和对CAP的取舍。

对于replication更详细的说明与解释，这里推荐[Distributed systems
for fun and profit](http://book.mixu.net/distsys/index.html)，后面，我会根据LedisDB的实际情况，详细的说明我在LedisDB里面使用的replication是如何实现的。

## BinLog

最开始的时候，Ledisdb采用的是类似MySQL通用binlog的replication机制，即通过binlog的filename + position来决定需要同步的数据。这套方式实现起来非常简单，但是仍然有一些不足，主要就在于hierarchical replication情况下如果master当掉，选择合适的slave提升为master是比较困难的。举个最简单的例子，假设A为master，B，C为slave，如果A当掉了，我们会在B，C里面选择同步数据最多的那个，但是是哪一个呢？这个问题，在MySQL的replication中也会碰到。

## MySQL GTID

在MySQL 5.6之后，引入了GTID（Global transaction ID）的概念来解决上述问题，它通过`Source:ID`的方式来在binlog里面表示一个唯一的transaction。Source为当前server的uuid，这个是全局唯一的，而ID则是该server内部的transaction ID（采用递增保证唯一）。具体到上面那个问题，采用GTID，如果A当掉了，我们只需要在B和C的binlog里面查找比较最后一个A这个uuid的transaction id的大小，譬如B的为uuid:10，而C的为uuid:30，那么铁定我们会选择C为新的master。

当然使用GTID也有相关的限制，譬如slave也必须写binlog等，但它仍然足够强大，解决了早期MySQL replication的时候一大摊子的棘手问题。但LedisDB并不准备使用，主要就在于应用场景没那么复杂的情况，我需要的是一个更加简单的解决方案。

## Google Global Transaction ID

早在MySQL的GTID之前，google的一个MySQL版本就已经使用了[global transaction id](https://code.google.com/p/google-mysql-tools/wiki/GlobalTransactionIds)，在binlog里面，它对于任何的transaction，使用了group id来唯一标示。group id是一个全局的递增ID，由master负责维护生成。当master当掉之后，我们只需要看slave的binlog里面谁的group id最大，那么那一个就是能被选为master了。

可以看到，这套方案非常简单，但是限制更多，譬如slave端的binlog只能由replication thread写入，不支持Multi-Masters，不支持circular replication等。但我觉得它已经足够简单高效，所以LedisDB准备参考它来实现。

## Raft

弄过分布式的童鞋应该都或多或少的接触过Paxos（至少我是没完全弄明白的），而[Raft](http://raftconsensus.github.io/)则号称是一个比Paxos简单得多的分布式一致性算法。

Raft通过replicated log来实现一致性，假设有A，B，C三台机器，A为Leader，B和C为follower，（其实也就是master和slave的概念）。A的任何更新，都必须首先写入Log（每个Log有一个LogID，唯一标示，全局递增），然后将其Log同步到至少Follower，然后才能在A上面提交更新。如果A当掉了，B和C重新选举，如果哪一台机器当前的LogID最大，则成为Leader。看到这里，是不是有了一种很熟悉的感觉？

LedisDB在支持consensus replication上面，参考了Raft的相关做法。

## 名词解释

在详细说明LedisDB replication的实现前，有必要解释一些关键字段。

+ LogID：log的唯一标示，由master负责生成维护，全局递增。
+ LastLogID：当前程序最新的logid，也就是记录着最后一次更新的log。
+ FirstLogID：当前程序最老的logid，之前的log已经被清除了。
+ CommitID：当前程序已经处理执行的log。譬如当前LastLogID为10，而CommitID为5，则还有6，7，8，9，10这几个log需要执行处理。如果CommitID = LastLogID，则证明程序已经处于最新状态，不再需要处理任何log了。

## LedisDB Replication

LedisDB的replication实现很简单，仍然是上面的例子，A，B，C三台机器，A为master，B和C为slave。

当master有任何更新，master会做如下事情：

1. 记录该更新到log，logid = LastLogID + 1，LastLogID = logid
2. 同步该log到slaves，等待slaves的确认返回，或者超时
3. 提交更新
4. 更新CommitID = logid

上面还需要考虑到错误处理的情况。

+ 如果1失败，记录错误日志，然后我们会认为该次更新操作失败，直接返回。
+ 如果3失败，不更新CommitID返回，因为这时候CommitID小于LastLogID，master进入read only模式，replication thread尝试执行log，如果能执行成功，则更新CommitID，变成可写模式。
+ 如果4失败，同上，因为LedisDB采用的是Row-Base Format的log格式，所以一次更新操作能够幂等多次执行。

对于slave

如果是首次同步，则进入全同步模式：

1. master生成一个snapshot，连同当前的LastLogID一起发送给slave。
2. slave收到该dump文件之后，load载入，同时更新CommitID为dump文件里面的LastLogID。

然后进入增量同步模式，如果slave已经有相关log，则直接进入增量同步模式。

在增量模式下面，slave向master发送sync命令，sync的参数为下一个需要同步的log，如果slave当前没有binlog（譬如上面提到的全同步情况），则logid = CommitID + 1， 否则logid = LastLogID + 1。

master收到sync请求之后，有如下处理情况：

+ sync的logid小于FirstLogID，master没有该log，slave收到该错误重新进入全同步模式。
+ master有该sync的log，于是将log发送给slave，slave收到之后保存，并再次发送sync获取下一个log，同时该次请求也作为ack告知master同步该log成功。
+ sync的log id已经大于LastLogID了，表明master和slave的状态已经到达一致，没有log可以同步了，slave将会等待新的log直到超时再次发送sync。

在slave端，对于接受到的log，由replication thread负责执行，并更新CommitID。

如果master当机，我们只需要选择具有最大LastLogID的那个slave为新的master就可以了。

## Limitation

总的来说，这套replication机制很简单，易于实现，但是仍然有许多限制。

+ 不支持Multi-Master，因为同时只能有一个地方进行全局LogID的生成。不过我真的很少见到Multi-Master这样的架构模式，即使在MySQL里面。
+ 不支持Circular-Replication，slave写入的log id不允许小于当前的LastLogID，这样才能保证只同步最新的log。
+ 没有自动master选举机制，不过我觉得放到外部去实现更好。

## Async/Sync Replication

LedisDB是支持强一致性的同步replication的，如果配置了该模式，那么master会等待slave同步完成log之后再提交更新，这样我们就能保证当master当机之后，一定有一台slave具有跟master一样的数据。但在实际中，可能因为网络环境等问题，master不可能一直等待slave同步完成log，所以通常都会有一个超时机制。所以从这点来看，我们仍然不能保证数据的强一致性。

使用同步replication机制会极大地降低master的写入性能，如果对数据一致性不敏感的业务，其实采用异步replication就可以了。

## Failover

LedisDB现在没有自动的failover机制，master当机之后，我们仍然需要外部的干预来选择合适的slave（具有最大LastLogID那个），提升为master，并将其他slave重新指向该master。后续考虑使用外部的keeper程序来处理。而对于keeper的单点问题，则考虑使用raft或者zookeeper来处理。

## 后记

虽然LedisDB现在已经支持replication，但仍然需要在生产环境中检验完善。

LedisDB是一个采用Go实现的高性能NoSQL，接口类似Redis，现在已经用于生产环境，欢迎大家使用。

[Official Website](http://ledisdb.com)

[Github](http://github.com/siddontang/ledisdb)