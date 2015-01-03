对于多数应用来说，MySQL都是作为最关键的数据存储中心的，所以，如何让MySQL提供HA服务，是我们不得不面对的一个问题。当master当机的时候，我们如何保证数据尽可能的不丢失，如何保证快速的获知master当机并进行相应的故障转移处理，都是需要我们好好思考的。这里，笔者将结合这段时间做的MySQL proxy以及toolsets相关工作，说说我们现阶段以及后续会在项目中采用的MySQL HA方案。

## Replication

要保证MySQL数据不丢失，replication是一个很好的解决方案，而MySQL也提供了一套强大的replication机制。只是我们需要知道，为了性能考量，replication是采用的asynchronous模式，也就是写入的数据并不会同步更新到slave上面，如果这时候master当机，我们仍然可能会面临数据丢失的风险。

为了解决这个问题，我们可以使用semi-synchronous replication，semi-synchronous replication的原理很简单，当master处理完一个事务，它会等待至少一个支持semi-synchronous的slave确认收到了该事件并将其写入relay-log之后，才会返回。这样即使master当机，最少也有一个slave获取到了完整的数据。

但是，semi-synchronous并不是100%的保证数据不会丢失，如果master在完成事务并将其发送给slave的时候崩溃，仍然可能造成数据丢失。只是相比于传统的异步复制，semi-synchronous replication能极大地提升数据安全。更为重要的是，它并不慢，MHA的作者都说他们在facebook的生产环境中使用了semi-synchronous（[这里](http://yoshinorimatsunobu.blogspot.ca/2014/04/semi-synchronous-replication-at-facebook.html)），所以我觉得真心没必要担心它的性能问题，除非你的业务量级已经完全超越了facebook或者google。

如果真的想完全保证数据不会丢失，现阶段一个比较好的办法就是使用[gelera](http://galeracluster.com/)，一个MySQL集群解决方案，它通过同时写三份的策略来保证数据不会丢失。笔者没有任何使用gelera的经验，只是知道业界已经有公司将其用于生产环境中，性能应该也不是问题。但gelera对MySQL代码侵入性较强，可能对某些有代码洁癖的同学来说不合适了:-)

我们还可以使用[drbd](http://drbd.linbit.com/)来实现MySQL数据复制，MySQL官方文档有一篇文档有详细[介绍](http://dev.mysql.com/doc/refman/5.6/en/ha-drbd.html)，但笔者并未采用这套方案，MHA的作者写了一些采用drdb的问题，[在这里](https://code.google.com/p/mysql-master-ha/wiki/Other_HA_Solutions#Pacemaker_+_DRBD)，仅供参考。

在后续的项目中，笔者会优先使用semi-synchronous replication的解决方案，如果数据真的非常重要，则会考虑使用gelera。

## Monitor

前面我们说了使用replication机制来保证master当机之后尽可能的数据不丢失，但是我们不能等到master当了几分钟才知道出现问题了。所以一套好的监控工具是必不可少的。

当master当掉之后，monitor能快速的检测到并做后续处理，譬如邮件通知管理员，或者通知守护程序快速进行failover。

通常，对于一个服务的监控，我们采用keepalived或者heartbeat的方式，这样当master当机之后，我们能很方便的切换到备机上面。但他们仍然不能很即时的检测到服务不可用。笔者的公司现阶段使用的是keepalived的方式，但后续笔者更倾向于使用zookeeper来解决整个MySQL集群的monitor以及failover。

对于任何一个MySQL实例，我们都有一个对应的agent程序，agent跟该MySQL实例放到同一台机器上面，并且定时的对MySQL实例发送ping命令检测其可用性，同时该agent通过ephemeral的方式挂载到zookeeper上面。这样，我们可以就能知道MySQL是否当机，主要有以下几种情况：

1. 机器当机，这样MySQL以及agent都会当掉，agent与zookeeper连接自然断开
2. MySQL当掉，agent发现ping不通，主动断开与zookeeper的连接
3. Agent当掉，但MySQL未当

上面三种情况，我们都可以认为MySQL机器出现了问题，并且zookeeper能够立即感知。agent与zookeeper断开了连接，zookeeper触发相应的children changed事件，监控到该事件的管控服务就可以做相应的处理。譬如如果是上面前两种情况，管控服务就能自动进行failover，但如果是第三种，则可能不做处理，等待机器上面crontab或者supersivord等相关服务自动重启agent。

使用zookeeper的好处在于它能很方便的对整个集群进行监控，并能即时的获取整个集群的变化信息并触发相应的事件通知感兴趣的服务，同时协调多个服务进行相关处理。而这些是keepalived或者heartbeat做不到或者做起来太麻烦的。

使用zookeeper的问题在于部署起来较为复杂，同时如果进行了failover，如何让应用程序获取到最新的数据库地址也是一个比较麻烦的问题。

对于部署问题，我们要保证一个MySQL搭配一个agent，幸好这年头有了docker，所以真心很简单。而对于第二个数据库地址更改的问题，其实并不是使用了zookeeper才会有的，我们可以通知应用动态更新配置信息，或者使用proxy来解决。

虽然zookeeper的好处很多，但如果你的业务不复杂，譬如只有一个master，一个slave，zookeeper可能并不是最好的选择，没准keepalived就够了。

## Failover

通过monitor，我们可以很方便的进行MySQL监控，同时在MySQL当机之后通知相应的服务做failover处理，假设现在有这样的一个MySQL集群，a为master，b，c为其slave，当a当掉之后，我们需要做failover，那么我们选择b，c中的哪一个作为新的master呢？

原则很简单，哪一个slave拥有最近最多的原master数据，就选哪一个作为新的master。我们可以通过`show slave status`这个命令来获知哪一个slave拥有最新的数据。我们只需要比较两个关键字段`Master_Log_File`以及`Read_Master_Log_Pos`，这两个值代表了slave读取到master哪一个binlog文件的哪一个位置，binlog的索引值越大，同时pos越大，则那一个slave就是能被提升为master。这里我们不讨论多个slave可能会被提升为master的情况。

在前面的例子中，假设b被提升为master了，我们需要将c重新指向新的master b来开始复制。我们通过`CHANGE MASTER TO`来重新设置c的master，但是我们怎么知道要从b的binlog的哪一个文件，哪一个position开始复制呢？

### GTID

为了解决这一个问题，MySQL 5.6之后引入了GTID的概念，即uuid:gid，uuid为MySQL server的uuid，是全局唯一的，而gid则是一个递增的事务id，通过这两个东西，我们就能唯一标示一个记录到binlog中的事务。使用GTID，我们就能非常方便的进行failover的处理。

仍然是前面的例子，假设b此时读取到的a最后一个GTID为`3E11FA47-71CA-11E1-9E33-C80AA9429562:23`，而c的为`3E11FA47-71CA-11E1-9E33-C80AA9429562:15`，当c指向新的master b的时候，我们通过GTID就可以知道，只要在b中的binlog中找到GTID为`3E11FA47-71CA-11E1-9E33-C80AA9429562:15`这个event，那么c就可以从它的下一个event的位置开始复制了。虽然查找binlog的方式仍然是顺序查找，稍显低效暴力，但比起我们自己去猜测哪一个filename和position，要方便太多了。

google很早也有了一个[Global Transaction ID](https://code.google.com/p/google-mysql-tools/wiki/GlobalTransactionIds)的补丁，不过只是使用的一个递增的整形，[LedisDB](https://github.com/siddontang/ledisdb)就借鉴了它的思路来实现failover，只不过google貌似现在也开始逐步迁移到MariaDB上面去了。

MariaDB的GTID实现跟MySQL 5.6是不一样的，这点其实比较麻烦，对于我的MySQL工具集[go-mysql](https://github.com/siddontang/go-mysql)来说，意味着要写两套不同的代码来处理GTID的情况了。后续是否支持MariaDB再看情况吧。

### Pseudo GTID

GTID虽然是一个好东西，但是仅限于MySQL 5.6+，当前仍然有大部分的业务使用的是5.6之前的版本，笔者的公司就是5.5的，而这些数据库至少长时间也不会升级到5.6的。所以我们仍然需要一套好的机制来选择master binlog的filename以及position。

最初，笔者打算研究[MHA](https://code.google.com/p/mysql-master-ha/)的实现，它采用的是首先复制relay log来补足缺失的event的方式，但笔者不怎么信任relay log，同时加之MHA采用的是perl，一个让我完全看不懂的语言，所以放弃了继续研究。

幸运的是，笔者遇到了[orchestrator](https://github.com/outbrain/orchestrator)这个项目，这真的是一个非常神奇的项目，它采用了一种[Pseudo GTID](http://code.openark.org/blog/mysql/refactoring-replication-topology-with-pseudo-gtid)的方式，核心代码就是这个

```
create database if not exists meta;

drop event if exists meta.create_pseudo_gtid_view_event;

delimiter ;;
create event if not exists
  meta.create_pseudo_gtid_view_event
  on schedule every 10 second starts current_timestamp
  on completion preserve
  enable
  do
    begin
      set @pseudo_gtid := uuid();
      set @_create_statement := concat('create or replace view meta.pseudo_gtid_view as select \'', @pseudo_gtid, '\' as pseudo_gtid_unique_val from dual');
      PREPARE st FROM @_create_statement;
      EXECUTE st;
      DEALLOCATE PREPARE st;
    end
;;

delimiter ;

set global event_scheduler := 1;
```

它在MySQL上面创建了一个事件，每隔1s，就将一个uuid写入到一个view里面，而这个是会记录到binlog中的，虽然我们仍然不能像GTID那样直接定位到一个event，但也能定位到一个1s的区间了，这样我们就能在很小的一个区间里面对比两个MySQL的binlog了。

继续上面的例子，假设c最后一次出现uuid的位置为s1，我们在b里面找到该uuid，位置为s2，然后依次对比后续的event，如果不一致，则可能出现了问题，停止复制。当遍历到c最后一个binlog event之后，我们就能得到此时b下一个event对应的filename以及position了，然后让c指向这个位置开始复制。

使用Pseudo GTID需要slave打开`log-slave-update`的选项，考虑到GTID也必须打开该选项，所以个人感觉完全可以接受。

后续，笔者自己实现的failover工具，将会采用这种Pseudo GTID的方式实现。

在《MySQL High Availability》这本书中，作者使用了另一种GTID的做法，每次commit的时候，需要在一个表里面记录gtid，然后就通过这个gtid来找到对应的位置信息，只是这种方式需要业务MySQL客户端的支持，笔者不很喜欢，就不采用了。

## 后记

MySQL HA一直是一个水比较深的领域，笔者仅仅列出了一些最近研究的东西，有些相关工具会尽量在[go-mysql](https://github.com/siddontang/go-mysql)中实现。