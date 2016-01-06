[ledisdb](https://github.com/siddontang/ledisdb)现在已经支持replication机制，为ledisdb的高可用做出了保障。

## 使用

假设master的ip为10.20.187.100，端口6380，slave的ip为10.20.187.101，端口为6380.

首先我们需要master打开binlog支持，在配置文件中指定：

    use_bin_log : true

在slave的机器上面我们可以通过配置文件指定slaveof开启replication，或者通过命令slaveof显示的开启或者关闭。

    slaveof 10.20.187.100 6380
    
ledisdb的replication机制参考了redis以及mysql的相关实现，下面简单说明。
    
## redis replication

redis的replication机制主要介绍在[这里](http://redis.io/topics/replication)，已经说明的很详细了。

+ slave向master发送sync命令
+ master将其当前的数据dump到一个文件，同时在内存中缓存新增的修改命令
+ 当数据dump完成，master就将其发送给slave
+ slave接受完成dump数据之后，将其本机先前的数据清空，然后在导入dump的数据
+ master再将先前缓存的命令发送给slave

在redis2.8之后，为了防止断线导致重新生成dump，redis增加了psync命令，在断线的时候master会记住当前的同步状态，这样下次就能进行断点续传了。

## mysql replication

mysql的replication主要是通过binlog的同步来完成的。在master的任何数据更新，都会写入binlog，至于binlog的格式这里不再累述。

假设binlog的basename为mysql，index文件名字为mysql-bin.index，该文件记录着当前所有的binlog文件。

binlog有max file size的配置，当binlog写入的的文件大小超过了该值，mysql就会生成一个新的binlog文件。当mysql服务重启的时候，也会生成一个新的binlog文件。

在Percona的mysql版本中，binlog还有一个max file num的设置，当binlog的文件数量超过了该值，mysql就会删除最早的binlog。

slave有一个master.info的文件，用以记录当前同步master的binlog的信息，主要就是当前同步的binlog文件名以及数据偏移位置，这样下次重新同步的时候就能从该位置继续进行。

slave同步的数据会写入relay log中，同时在后台有另一个线程将relay log的数据存入mysql。

因为master的binlog可能删除，slave同步的时候可能会出现binlog丢失的情况，mysql通过[dump+binlog](http://dev.mysql.com/doc/refman/5.0/en/backup-policy.html)的方式解决，其实也就是slave完全的dump master数据，在生成的dump中也同时会记录当前的binlog信息，便于下次继续同步。

## ledisdb replication

ledisdb的replication机制参考了redis以及mysql，支持fullsync以及增量sync。

master没有采用aof机制，而是使用了binlog，通过指定max file size以及max file num用来控制binlog的总体大小，这样我就无需关心aof文件持续增大需要重新rewrite的过程了。

binlog文件名格式如下：

    ledis-bin.0000001
    ledis-bin.0000002
    
binlog文件名的后缀采用数字递增，后续我们使用index来表示。

slave端也有一个master.info文件，因为ledisdb会严格的保证binlog文件后缀的递增，所以我们只需要记录当前同步的binlog文件后缀的index即可。

整个replication流程如下：

- 当首次同步或者记录的binlog信息因为master端binlog删除导致不一致的时候，slave会发送fullsync进行全同步。
- master收到fullsync信息之后，会将当前的数据以及binlog信息dump到文件，并将其发送给slave。
- slave接受完成整个dump文件之后，清空所有数据，同时将dump的数据导入leveldb，并保存当前dump的binlog信息。
- slave通过sync命令进行增量同步，sync命令格式如下：

        sync binlog-index binlog-pos
   
   master通过index定位到指定的binlog文件，并seek至pos位置，将其后面的binlog数据发送给slave。
- slave接收到binlog数据，导入leveldb，如果sync没有收到任何新增数据，1s之后再次sync。

对于最后一点，最主要就是一个问题，即master新增的binlog如何让slave进行同步。对于这点无非就是两种模型，push和pull。

对于push来说，任何新增的数据都能非常及时的通知slave去获取，而pull模型为了性能考虑，不可能太过于频繁的去轮询，略有延时。

mysql采用的是push + pull的模式，当binlog有更新的时候，仅仅通知slave有了更新，slave则是通过pull拉取实际的数据。但是为了支持push，master必须得维持slave的一些状态信息，这稍微又增加了一点复杂度。

ledisdb采用了非常简单的一种方式，定时pull，使用1s的间隔，这样既不会因为轮询太过频繁导致性能开销增大，同时也能最大限度的减少当机数据丢失的风险。

## 总结

ledisdb的replication机制才刚刚完成，后续还有很多需要完善，但足以使其成为一个高可用的nosql选择了。

ledisdb的网址在这里[https://github.com/siddontang/ledisdb](https://github.com/siddontang/ledisdb)，希望感兴趣的童鞋共同参与。