## Why Elasticsearch？

由于需要提升项目的搜索质量，最近研究了一下Elasticsearch，一款非常优秀的分布式搜索程序。最开始的一些笔记放到[github](https://github.com/siddontang/elasticsearch-note)，这里只是归纳总结一下。

首先，为什么要使用Elasticsearch？最开始的时候，我们的项目仅仅使用MySQL进行简单的搜索，然后一个不能索引的like语句，直接拉低MySQL的性能。后来，我们曾考虑过sphinx，并且sphinx也在之前的项目中成功实施过，但想想现在的数据量级，多台MySQL，以及搜索服务本身HA，还有后续扩容的问题，我们觉得sphinx并不是一个最优的选择。于是自然将目光放到了Elasticsearch上面。

根据官网自己的介绍，Elasticsearch是一个分布式搜索服务，提供Restful API，底层基于Lucene，采用多shard的方式保证数据安全，并且提供自动resharding的功能，加之github等大型的站点也采用Elasticsearch作为其搜索服务，我们决定在项目中使用Elasticsearch。

对于Elasticsearch，如果要在项目中使用，需要解决如下问题：

1. 索引，对于需要搜索的数据，如何建立合适的索引，还需要根据特定的语言使用不同的analyzer等。
2. 搜索，Elasticsearch提供了非常强大的搜索功能，如何写出高效的搜索语句？
3. 数据源，我们所有的数据是存放到MySQL的，MySQL是唯一数据源，如何将MySQL的数据导入到Elasticsearch？

对于1和2，因为我们的数据都是从MySQL生成，index的field是固定的，主要做的工作就是根据业务场景设计好对应的mapping以及search语句就可以了，当然实际不可能这么简单，需要我们不断的调优。

而对于3，则是需要一个工具将MySQL的数据导入Elasticsearch，因为我们对搜索实时性要求很高，所以需要将MySQL的增量数据实时导入，笔者唯一能想到的就是通过row based binlog来完成。而近段时间的工作，也就是实现一个MySQL增量同步到Elasticsearch的服务。

## Lucene

Elasticsearch底层是基于Lucene的，Lucene是一款优秀的搜索lib，当然，笔者以前仍然没有接触使用过。:-)

Lucene关键概念：

+ Document：用来索引和搜索的主要数据源，包含一个或者多个Field，而这些Field则包含我们跟Lucene交互的数据。
+ Field：Document的一个组成部分，有两个部分组成，name和value。
+ Term：不可分割的单词，搜索最小单元。
+ Token：一个Term呈现方式，包含这个Term的内容，在文档中的起始位置，以及类型。

Lucene使用[Inverted index](http://en.wikipedia.org/wiki/Inverted_index)来存储term在document中位置的映射关系。
譬如如下文档：

- Elasticsearch Server 1.0 （document 1）
- Mastring Elasticsearch （document 2）
- Apache Solr 4 Cookbook （document 3）

使用inverted index存储，一个简单地映射关系：

|Term|Count|Docuemnt|
|----|-----|--------|
|1.0|1|\<1\>|
|4|1|\<3\>|
|Apache|1|\<3\>|
|Cookbook|1|\<3\>|
|Elasticsearch|2|\<1\>.\<2\>|
|Mastering|1|\<2\>|
|Server|1|\<1\>|
|Solr|1|\<3\>|

对于上面例子，我们首先通过分词算法将一个文档切分成一个一个的token，再得到该token与document的映射关系，并记录token出现的总次数。这样就得到了一个简单的inverted index。

## Elasticsearch关键概念

要使用Elasticsearch，笔者认为，只需要理解几个基本概念就可以了。

在数据层面，主要有：

+ Index：Elasticsearch用来存储数据的逻辑区域，它类似于关系型数据库中的table概念。一个index可以在一个或者多个shard上面，同时一个shard也可能会有多个replicas。
+ Document：Elasticsearch里面存储的实体数据，类似于关系数据中一个table里面的一行数据。
document由多个field组成，不同的document里面同名的field一定具有相同的类型。document里面field可以重复出现，也就是一个field会有多个值，即multivalued。
+ Document type：为了查询需要，一个index可能会有多种document，也就是document type，但需要注意，不同document里面同名的field一定要是相同类型的。
+ Mapping：存储field的相关映射信息，不同document type会有不同的mapping。

对于熟悉MySQL的童鞋，我们只需要大概认为Index就是一个table，document就是一行数据，field就是table的column，mapping就是table的定义就可以了。

Document type这个概念其实最开始也把笔者给弄糊涂了，其实它就是为了更好的查询，举个简单的例子，一个index，可能一部分数据我们想使用一种查询方式，而另一部分数据我们想使用另一种查询方式，于是就有了两种type了。不过这种情况应该在我们的项目中不会出现，所以通常一个index下面仅会有一个type。

在服务层面，主要有：

+ Node: 一个server实例。
+ Cluster：多个node组成cluster。
+ Shard：数据分片，一个index可能会存在于多个shards，不同shards可能在不同nodes。
+ Replica：shard的备份，有一个primary shard，其余的叫做replica shards。

Elasticsearch之所以能动态resharding，主要在于它最开始就预先分配了多个shards（貌似是1024），然后以shard为单位进行数据迁移。这个做法其实在分布式领域非常的普遍，[codis](github.com/wandoulabs/codis)就是使用了1024个slot来进行数据迁移。

因为任意一个index都可配置多个replica，通过冗余备份的方式保证了数据的安全性，同时replica也能分担读压力，类似于MySQL中的slave。

## Restful API

Elasticsearch提供了Restful API，使用json格式，这使得它非常利于与外部交互，虽然Elasticsearch的客户端很多，但笔者仍然很容易的就写出了一个简易客户端用于项目中，再次印证了Elasticsearch的使用真心很容易。

Restful的接口很简单，一个url表示一个特定的资源，譬如`/blog/article/1`，就表示一个index为blog，type为aritcle，id为1的document。

而我们使用http标准method来操作这些资源，POST新增，PUT更新，GET获取，DELETE删除，HEAD判断是否存在。

这里，友情推荐[httpie](https://github.com/jakubroztocil/httpie)，一个非常强大的http工具，个人感觉比curl还用，几乎是命令行调试Elasticsearch的绝配。

一些使用httpie的例子:

```
# create
http POST :9200/blog/article/1 title="hello elasticsearch" tags:='["elasticsearch"]'

# get
http GET :9200/blog/article/1

# update
http PUT :9200/blog/article/1 title="hello elasticsearch" tags:='["elasticsearch", "hello"]'

# delete
http DELETE :9200/blog/article/1

# exists
http HEAD :9200/blog/article/1

```

## 索引和搜索

虽然Elasticsearch能自动判断field类型并建立合适的索引，但笔者仍然推荐自己设置相关索引规则，这样才能更好为后续的搜索服务。

我们通过定制mapping的方式来设置不同field的索引规则。

而对于搜索，Elasticsearch提供了太多的搜索选项，就不一一概述了。

索引和搜索是Elasticsearch非常重要的两个方面，直接关系到产品的搜索体验，但笔者现阶段也仅仅是大概了解了一点，后续在详细介绍。

## 同步MySQL数据 

Elasticsearch是很强大，但要建立在有足量数据情况下面。我们的数据都在MySQL上面，所以如何将MySQL的数据导入Elasticsearch就是笔者最近研究的东西了。

虽然现在有一些实现，譬如[elasticsearch-river-jdbc](https://github.com/jprante/elasticsearch-river-jdbc)，或者[elasticsearch-river-mysql](https://github.com/scharron/elasticsearch-river-mysql)，但笔者并不打算使用。

elasticsearch-river-jdbc的功能是很强大，但并没有很好的支持增量数据更新的问题，它需要对应的表只增不减，而这个几乎在项目中是不可能办到的。

elasticsearch-river-mysql倒是做的很不错，采用了[python-mysql-replication](https://github.com/noplay/python-mysql-replication)来通过binlog获取变更的数据，进行增量更新，但它貌似处理MySQL dump数据导入的问题，不过这个笔者真的好好确认一下？话说，python-mysql-replication笔者还提交过[pull](https://github.com/noplay/python-mysql-replication/pull/103)解决了minimal row image的问题，所以对elasticsearch-river-mysql这个项目很有好感。只是笔者决定自己写一个出来。

为什么笔者决定自己写一个，不是因为笔者喜欢造轮子，主要原因在于对于这种MySQL syncer服务（增量获取MySQL数据更新到相关系统），我们不光可以用到Elasticsearch上面，而且还能用到其他服务，譬如cache上面。所以笔者其实想实现的是一个通用MySQL syncer组件，只是现在主要关注Elasticsearch罢了。

项目代码在这里[go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch)，仍然处于开发状态，预计下周能基本完成。

go-mysql-elasticsearch的原理很简单，首先使用mysqldump获取当前MySQL的数据，然后在通过此时binlog的name和position获取增量数据。

一些限制：

+ binlog一定要变成row-based format格式，其实我们并不需要担心这种格式的binlog占用太多的硬盘空间，MySQL 5.6之后GTID模式都推荐使用row-based format了，而且通常我们都会把控SQL语句质量，不允许一次性更改过多行数据的。
+ 需要同步的table最好是innodb引擎，这样mysqldump的时候才不会阻碍写操作。
+ 需要同步的table一定要有主键，好吧，如果一个table没有主键，笔者真心会怀疑设计这个table的同学编程水平了。多列主键也是不推荐的，笔者现阶段不打算支持。
+ 一定别动态更改需要同步的table结构，Elasticsearch只能支持动态增加field，并不支持动态删除和更改field。通常来说，如果涉及到alter table，很多时候已经证明前期设计的不合理以及对于未来扩展的预估不足了。

更详细的说明，等到笔者完成了go-mysql-elasticsearch的开发，再进行补充。

## 总结

最近一周，笔者花了不少时间在Elasticsearch上面，现在算是基本入门了。其实笔者觉得，对于一门不懂的技术，找一份靠谱的资料（官方文档或者入门书籍），蛋疼的对着资料敲一遍代码，不懂的再问google，最后在将其用到实际项目，这门技术就算是初步掌握了，当然精通还得在下点功夫。

现在笔者只是觉得Elasticsearch很美好，上线之后铁定会有坑的，那时候只能慢慢填了。话说，笔者是不是要学习下java了，省的到时候看不懂代码就惨了。:-)