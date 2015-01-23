简单记录一次MySQL优化经历。

我们需要在一张表里面删除某种类型的数据，大概的表结构类似这样:

```
CREATE TABLE t (
    id int,
    tp enum ("t1", "t2"),
    primary key(id)
) ENGINE=INNODB;
```

假设我们需要删除t2，语句可能是这样`delete from t where tp = "t2"`，这样没啥问题，但我们这张表有5亿数据，好吧，真的是5亿，所以以后别再跟我说MySQL表存储百万级别数据就要分表了，百万太小case了。

这事情我交给了一个小盆友去帮我搞定，然后他最开始写出了如下的语句`delete from t where tp = "t2" limit 1000`，使用limit来限制一次删除的个数，可以了，不过这有个很严重的问题，就是越往后，随着t2类型的减少，我们几乎都是全表遍历来删除，所以总的应该是O(n*n)的开销。

于是我让他考虑主键，每次操作的时候，记录当前最大的主键，这样下次就可以从这个主键之后开始删除了，首先 `select id from t where id > last_max_select_id and tp = "t2" limit 1000`，然后`delete from t where id in (ids)`，虽然这次优化采用了两条语句，但是通过主键，我们只需要遍历一次表就可以了，总的来说，性能要快的。

但是，实际测试的时候，我们却发现，select这条语句耗时将近30s，太慢了。虽然我们使用了主键，但是MySQL仍然需要不停的读取数据判断条件，加之t2类型的数据在表里面比较少量，所以为了limit 1000这个条件，MySQL需要持续的进行IO读取操作，结果自然是太慢了。

想清楚了这个，其实就好优化了，我们只需要让条件判断在应用层做，MySQL只查询数据返回，语句就是 `select id, tp from t where id > last_max_select_id limit 1000`，得到结果集之后，自行判断需要删除的id，然后delete。看似我们需要额外处理逻辑，并且网络开销也增大了，但MySQL只是简单的IO读取，非常快，总的来说，性能提升很显著。

最后执行，很happy的是，非常快速的就删完了相关数据，而select的查询时间消耗几乎忽略不计。