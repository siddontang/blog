# 介绍

这段时间在重构server，尤其是重新设计表结构以满足功能需求以及后续的性能需求。因为我们使用的是mysql，所以为了支持更多的并发访问，对大数据量的表设计一些index是必不可少的，而现在看我当时设计的index，又发现了很多不足的地方。同时又因为深入重新在学习了解了一些index知识，觉得有必要记录一下，以免自己再犯同样的错误。

假设有如下的表结构，后续所有的例子都通过该表来说明：

    table tbl,
    fileid int,
    groupid int,
    opver int,
    name varchar(1024),
    entid int,
    primary key (fileid),
    engine = innodb

# 聚集索引

因为使用的mysql engine是innodb，所以对于主键使用的是聚簇索引，对于聚簇索引我们知道它是吧数据存放到index里面的，也就是通过主键查询就能直接定位到数据，非常方便。

以前我受到的教育就是，select \*是邪恶的，因为会取出所有列的数据。诚然，对于数据的获取，我们是按需索取，这样就能极大的减少网络传输的消耗。正因为以前有这样的认识，我自然认为innodb在server端io数据读取的时候也是按需索取。但其实不是这样。

在innodb里面，数据是按照页来存放的，通常一页为16k，每一页至少存放2条数据，也就是说，每一条数据在最大为8k。这里有童鞋就可能疑惑了，那如果我表结构里面有blob，text这种类型的肿么办？对于这种大数据类型的，innodb会将其存放到一个overflow page里面去，然后数据页只存放对应的指针。同样，innodb也是按照页来读取数据，也就是说一次读取16k。

对于上面我们的表结构，是完全能够存放到同一个page里面的，也就是说，我们即使select的时候没有选择对应的列，在innodb层面也仍然会将该数据读出，也就说无论怎样，io消耗是一样的。具体可以参考[这篇](http://www.zhaokunyao.com/archives/1558)。

这里在谈谈主键id类型选择的问题，因为innodb采用的是b+tree来进行数据的存储，网上对于是否采用自增id或者guid这种争论满天飞，现在我的想法如下：

- guid会导致主键过于分散，以至于在数据进行插入，删除，更新的时候会进行b+tree的频繁分裂，同时读取数据的时候因为过于分散会导致io随即读取，性能不高。
- 自增id虽然能让io进行顺序读取，但是如果服务器数据量太大造成分表了，对于每张表的auto increment的初始值维护也是一件很烦的事情。
- 鉴于上面那种情况，我们现在考虑的做法就是提供一个id生成器，用来保证主键的顺序递增，同时又保证全局的完全唯一。这个id生成器很容易，使用redis的incr就可以搞定，而且还可以通过incrby进行批量申请，性能妥妥的。

# 联合索引

一般为了查询方便，我们有可能在多列上面建立联合索引，譬如在\(groupid, opver\)上面，那么我就可以很方便的使用如下索引：

    select * from tbl where groupid in (1,2,3)
    select * from tbl where groupid = 1 and opver > 10 and opver < 100
    select * from tbl where groupid = 1 and opver > 10 order by opver desc limit 10

在上面那些查询语句立马，index都能很好的工作，然后我就自认为下面这种的也行：

    select * from tbl where groupid in (1,2,3) and opver > 10 order by opver desc limit 10

对于这个查询，我自然想到的就是首先mysql会通过索引选出groupid为1，2，3的，然后再在这个结果集里面选出opver满足的进行排序。可真的是这样吗。使用explain之后，才发现，在extra那一栏生生的出现了using filesort。也就是说mysql并没有对opver使用索引进行order by，而是使用了filesort，这里我们的索引在groupid之后就无效了。

其实这种情况可以用通用的情况来归纳，在一个组合索引里面，如果出现了多个范围查询，那么mysql不可能为每一个范围都使用索引，一般碰到第一个范围之后就会停止索引工作了。

这里给了我一个深刻的教训，就是设计好表结构以及index之后，不要自认为就能按照我想的方式工作了，最好需要explain一下，看看mysql到底是怎么工作的，不然出了性能瓶颈都不知道。

# 覆盖索引

当我们需要查询的数据通过索引就能查到的时候，这种索引就叫做覆盖索引。譬如下面这些:

    select groupid, opver from tbl where groupid = 1 and opver > 10;

    select fileid, groupid, opver from tbl where groupid = 1 and opver > 10 order by opver limit 10

因为我们在(groupid, opver)上面建立了索引，那么当我们查询的所有列包含在该索引里面的时候，我们就可以通过覆盖索引直接找到。通过explain可以看到extra里面有using index，表明使用了覆盖索引。

对于第二个例子，为什么fileid也可以使用覆盖索引呢，因为对于innodb来说，非聚簇索引保存的是主键id，也就是fileid，所以通过索引也能够直接找到fileid。

使用覆盖索引的好处是很明显的，数据只需要通过索引就能获取，而不需要通过主键在进行随机的io读取。一个很简单的例子。

    select entid from t1 order by opver limit 100000, 10;

    select entid from t1 inner join (select fileid from t1 order by opver limit 100000, 10) as ac using(fileid)

因为我们需要查询entid，而没有任何一个索引能覆盖entid，这里我们给opver单独建立一个索引。对于第一个查询，mysql因为需要获取entid，所以会通过fileid查到实际的数据并取出，在进行排序，同时因为limit的跨度很大，mysql会丢弃很多数据，所以导致该查询会很慢。

而对于第二个查询，在join立马，我们只是通过覆盖索引找到了fileid，而不需要通过fileid去随机io读取实际数据，取出所有的fileid之后，在取出了entid。虽然该查询有一个join操作，但是因为join的性能很高，所以比第一个查询快很多。

关于索引覆盖，网上有很多关于这个的介绍，譬如[这个](http://hi.baidu.com/shinegun/item/1f36f5517bd8b4dfd48bac2d)，总之用好了覆盖索引，是能极大提升效率的，但是也不可能为了覆盖索引去建立覆盖索引，如果索引泛滥了，也是一件很头疼的事情。

# filesort

对于先前联合索引遇到的问题，我发现如果不重新设计整个的查询机制，按照现有做法是完全不能避免filesort的。因为我们的业务逻辑就是需要查询一批groupid里面，opver大于某个值的所有数据。既然避免不了，那我们就可以看看到底性能高不高。

mysql对于filesort，有两种处理方式，一种是单路排序，一种是双路排序。单路排序直接就是取出选择的字段，然后在sort buffer中排序。而双路排序则是取出排序字段以及主键，排序完成之后在通过主键获取实际的数据。可以看出，双路排序会有两次io操作，自然性能会差一点。

早期mysql采用的是双路排序，现在采用的是单路，但是如果我们需要排序的数据超过了配置的sort buffer空间，或者需要排序的单行数据长度大于max_length_for_sort_data，mysql仍然会使用单路排序。mysql默认max_length_for_sort_data为1024，这个需要根据实际数据长度进行配置。

不过如果能把mysql的硬盘换成ssd，或者把内存加到100g以上，我觉得这个filesort也还真不成问题了。

# 后续

这几天对于mysql索引的研究就先记录一下，感觉对mysql的理解还需要加深，后续如果还有什么新的感想也会在这里记录。同时bs一下自己，号称用了mysql这么多年，到现在了还有很多东西没有领会，是时候开始好好深入研究了。