## Why

最开始的时候，[go-mysql][1]只是简单的抽象[mixer][2]的代码，提供一个基本的mysql driver以及proxy framework，但做到后面，笔者突然觉得，既然研究了这么久mysql client/server protocol，干脆顺带把replication protocol也给弄明白算了。现在想想，幸好当初决定实现了replication的支持，不然后续[go-mysql-elasticsearch][3]这个自动同步MySQL到Elasticsearch的工具就不可能在短时间完成。

其实MySQL replication protocol很简单，client向server发送一个MySQL binlog dump的命令，server就会源源不断的给client发送一个接一个的binlog event了。

## Register

首先，我们需要伪造一个slave，向master注册，这样master才会发送binlog event。注册很简单，就是向master发送COM_REGISTER_SLAVE命令，带上slave相关信息。这里需要注意，因为在MySQL的replication topology中，都需要使用一个唯一的server id来区别标示不同的server实例，所以这里我们伪造的slave也需要一个唯一的server id。

## Binlog dump

最开始的时候，MySQL只支持一种binlog dump方式，也就是指定binlog filename + position，向master发送COM_BINLOG_DUMP命令。在发送dump命令的时候，我们可以指定flag为BINLOG_DUMP_NON_BLOCK，这样master在没有可发送的binlog event之后，就会返回一个EOF package。不过通常对于slave来说，一直把连接挂着可能更好，这样能更及时收到新产生的binlog event。

在MySQL 5.6之后，支持了另一种dump方式，也就是GTID dump，通过发送COM_BINLOG_DUMP_GTID命令实现，需要带上的是相应的GTID信息，不过笔者觉得，如果只是单纯的实现一个能同步binlog的工具，使用最原始的binlog filename + position就够了，毕竟我们不是MySQL，解析GTID还是稍显麻烦的。这里，顺带吐槽一下MySQL internal文档，里面关于GTID encode的格式说明竟然是错误的，文档格式如下:

```
4                n_sids
  for n_sids {
string[16]       SID
8                n_intervals
    for n_intervals {
8                start (signed)
8                end (signed)
    }
```
但实际坑爹的是n_sids的长度是8个字节。这个错误可以算是血的教训，笔者当时debug了很久都没发现为啥GTID dump一直出错，直到笔者查看了MySQL的源码。

MariaDB虽然也引入了GTID，但是并没有提供一个类似MySQL的GTID dump命令，仍是使用的COM_BINLOG_DUMP命令，不过稍微需要额外设置一些session variable，譬如将设置slave_connect_state为当前已经完成的GTID，这样master就能知道下一个event从哪里发送了。

## Binlog Event

对于一个binlog event来说，它分为三个部分，header，post-header以及payload。但实际笔者在处理event的时候，把post-header和payload当成了一个整体body。

MySQL的binlog event有很多版本，但这里笔者只关心version 4的，也就是从MySQL 5.1.x之后支持的版本。而且笔者也只支持这个版本的event解析，首先是不想写过多的兼容代码，另一个更主要的原因就在于现在几乎都没有人使用低版本的MySQL了。

Binlog event的header格式如下：

```
4              timestamp
1              event type
4              server-id
4              event-size
4              log pos
2              flags
```

header的长度固定为19，event type用来标识这个event的类型，event size则是该event包括header的整体长度，而log pos则是下一个event所在的位置。

在v4版本的binlog文件中，第一个event就是FORMAT_DESCRIPTION_EVENT，格式为:

```
2                binlog-version
string[50]       mysql-server version
4                create timestamp
1                event header length
string[p]        event type header lengths
```
我们很要关注的就是event type header lengths这个字段，它保存了不同event的post-header长度，通常我们都不需要关注这个值，但是在解析后面非常重要的row event的时候，就需要它来判断TableID的长度了。这个后续在说明。

而binlog文件的结尾，通常（只要master不当机）就是ROTATE_EVENT或者STOP_EVENT。这里我们重点关注ROTATE_EVENT，格式如下:

```
Post-header
8              position
Payload
string[p]      name of the next binlog
```
它里面其实就是标明下一个event所在的binlog filename和position。这里需要注意，当slave发送binlog dump之后，master首先会发送一个ROTATE_EVENT，用来告知slave下一个event所在位置，然后才跟着FORMAT_DESCRIPTION_EVENT。

其实我们可以看到，binlog event的格式很简单，文档都有着详细的说明。通常来说，我们仅仅需要关注几种特定类型的event，所以只需要写出这几种event的解析代码就可以了，剩下的完全可以跳过。

## Row Based Replication

如果真要说处理binlog event有啥复杂的，那铁定属于row based replication相关的rows event了，对于一个rows event来说，它记录了每一行数据的变化情况，而对于外部来说，是需要准确的知道这一行数据到底如何变化的，所以我们需要获取到改行每一列的值。而如何解析相关的数据，是非常复杂的。笔者也是看了很久MySQL，MariaDB源码，以及[mysql-python-replication][4]的实现，才最终搞定了这个个人觉得最困难的部分。

//todo......

[1]: https://github.com/siddontang/go-mysql  "A MySQL toolset"
[2]: https://github.com/siddontang/mixer  "A MySQL Proxy"
[3]: https://github.com/siddontang/go-mysql-elasticsearch  "Sync MySQL into Elasticsearch"
[4]: https://github.com/noplay/python-mysql-replication "Pure python for MySQL replication protocol"
