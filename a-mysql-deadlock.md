这里简单记录一个最近碰到的MySQL死锁bug，假设有如下表结构：

```
mysql> show create table tt \G;
*************************** 1. row ***************************
       Table: tt
Create Table: CREATE TABLE `tt` (
  `id` int(11) NOT NULL DEFAULT '0',
  `fileid` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `fileid` (`fileid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8
1 row in set (0.00 sec)
```

启动三个shell，连接MySQL，然后`begin`开启一个事务，各个shell分别执行对应的更新语句，

shell 1：
```
shell 1> update tt set id = 2 where fileid = 1;
```
shell 2：
```
shell 2> update tt set id = 3 where fileid = 1;
```
shell 3：
```
shell 3> update tt set id = 4 where fileid = 1;
```

假设shell 1先执行，这时候2和3会block，然后shell 1 commit提交，我们发现shell 2执行成功，但是3出现死锁错误，通过`show engine innodb status`我们得到如下死锁信息:

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2015-01-23 14:24:16 10ceed000
*** (1) TRANSACTION:
TRANSACTION 24897, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 360, 1 row lock(s)
MySQL thread id 8, OS thread handle 0x10cea5000, query id 138 127.0.0.1 root updating
update tt set id = 4 where fileid = 1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 495 page no 4 n bits 72 index `fileid` of table `test`.`tt` trx id 24897 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000001; asc     ;;
 1: len 4; hex 80000002; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 24896, ACTIVE 8 sec updating or deleting
mysql tables in use 1, locked 1
4 lock struct(s), heap size 1184, 3 row lock(s), undo log entries 2
MySQL thread id 7, OS thread handle 0x10ceed000, query id 136 127.0.0.1 root updating
update tt set id = 3 where fileid = 1
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 495 page no 4 n bits 72 index `fileid` of table `test`.`tt` trx id 24896 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000001; asc     ;;
 1: len 4; hex 80000002; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 495 page no 4 n bits 72 index `fileid` of table `test`.`tt` trx id 24896 lock mode S waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 2; compact format; info bits 32
 0: len 4; hex 80000001; asc     ;;
 1: len 4; hex 80000002; asc     ;;

*** WE ROLL BACK TRANSACTION (1)
------------

```

刚开始碰到这个死锁问题，真心觉得很奇怪，每个事务一条语句，通过一个唯一索引去更新同一条记录，正常来说完全不可能发生死锁，但确确实实发生了。笔者百思不得其解，幸好有google，然后搜到了这篇，[一个最不可思议的MySQL死锁分析](http://hedengcheng.com/?p=844)，虽然触发情况不一样，但是死锁原理都应该类似的，后续如果有精力，笔者将好好深入研究一下。

顺带再说一下，[MySQL 加锁处理分析](http://hedengcheng.com/?p=771)这篇文章也是干活满满，这两篇加起来深入理解了，对MySQL的deadlock就会有一个很全面的认识了。
