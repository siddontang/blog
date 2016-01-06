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

MariaDB虽然也引入了GTID，但是并没有提供一个类似MySQL的GTID dump命令，仍是使用的COM_BINLOG_DUMP命令，不过稍微需要额外设置一些session variable，譬如要设置slave_connect_state为当前已经完成的GTID，这样master就能知道下一个event从哪里发送了。

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
我们需要关注的就是event type header length这个字段，它保存了不同event的post-header长度，通常我们都不需要关注这个值，但是在解析后面非常重要的ROWS_EVENT的时候，就需要它来判断TableID的长度了。这个后续在说明。

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

如果真要说处理binlog event有啥复杂的，那铁定属于row based replication相关的ROWS_EVENT了，对于一个ROWS_EVENT来说，它记录了每一行数据的变化情况，而对于外部来说，是需要准确的知道这一行数据到底如何变化的，所以我们需要获取到该行每一列的值。而如何解析相关的数据，是非常复杂的。笔者也是看了很久MySQL，MariaDB源码，以及[mysql-python-replication][4]的实现，才最终搞定了这个个人觉得最困难的部分。

在详细说明ROWS_EVENT之前，我们先来看看TABLE_MAP_EVENT，该event记录的是某个table一些相关信息，格式如下:

```
post-header:
    if post_header_len == 6 {
  4              table id
    } else {
  6              table id
    }
  2              flags

payload:
  1              schema name length
  string         schema name
  1              [00]
  1              table name length
  string         table name
  1              [00]
  lenenc-int     column-count
  string.var_len [length=$column-count] column-def
  lenenc-str     column-meta-def
  n              NULL-bitmask, length: (column-count + 8) / 7
```

table id需要根据post_header_len来判断字节长度，而post_header_len就是存放到FORMAT_DESCRIPTION_EVENT里面的。这里需要注意，虽然我们可以用table id来代表一个特定的table，但是因为alter table或者rotate binlog event等原因，master会改变某个table的table id，所以我们在外部不能使用这个table id来索引某个table。

TABLE_MAP_EVENT最需要关注的就是里面的column meta信息，后续我们解析ROWS_EVENT的时候会根据这个来处理不同数据类型的数据。column def则定义了每个列的类型。

ROWS_EVENT包含了insert，update以及delete三种event，并且有v0，v1以及v2三个版本。

ROWS_EVENT的格式很复杂，如下：

```
header:
  if post_header_len == 6 {
4                    table id
  } else {
6                    table id
  }
2                    flags
  if version == 2 {
2                    extra-data-length
string.var_len       extra-data
  }

body:
lenenc_int           number of columns
string.var_len       columns-present-bitmap1, length: (num of columns+7)/8
  if UPDATE_ROWS_EVENTv1 or v2 {
string.var_len       columns-present-bitmap2, length: (num of columns+7)/8
  }

rows:
string.var_len       nul-bitmap, length (bits set in 'columns-present-bitmap1'+7)/8
string.var_len       value of each field as defined in table-map
  if UPDATE_ROWS_EVENTv1 or v2 {
string.var_len       nul-bitmap, length (bits set in 'columns-present-bitmap2'+7)/8
string.var_len       value of each field as defined in table-map
  }
  ... repeat rows until event-end
```

ROWS_EVENT的table id跟TABLE_MAP_EVENT一样，虽然table id可能变化，但是ROWS_EVENT和TABLE_MAP_EVENT的table id是能保证一致的，所以我们也是通过这个来找到对应的TABLE_MAP_EVENT。

为了节省空间，ROWS_EVENT里面对于各列状态都是采用bitmap的方式来处理的。

首先我们需要得到columns present bitmap的数据，这个值用来表示当前列的一些状态，如果没有设置，也就是某列对应的bit为0，表明该ROWS_EVENT里面没有该列的数据，外部直接使用null代替就成了。

然后就是null bitmap，这个用来表明一行实际的数据里面有哪些列是null的，这里最坑爹的是null bitmap的计算方式并不是`(num of columns+7)/8`，也就是MySQL计算bitmap最通用的方式，而是通过columns present bitmap的bits set个数来计算的，这个坑真的很大，为啥要这么设计，最主要的原因就在于MySQL 5.6之后binlog row image的格式增加了minimal和noblob，尤其是minimal，update的时候只会记录相应更改字段的数据，譬如我一行有16列，那么用2个byte就能搞定null bitmap了，但是如果这时候只有第一列更新了数据，其实我们只需要使用1个byte就能记录了，因为后面的铁定全为0，就不需要额外空间存放了，不过话说真有必要这么省空间吗？

null bitmap的计算需要通过columns present bitmap的bits set计算，bits set其实也很好理解，就是一个byte按照二进制展示的时候1的个数，譬如1的bits set就是1，而3的bits set就是2，而255的bits set就是8了。

好了，得到了present bitmap以及null bitmap之后，我们就能实际解析这行对应的列数据了，对于每一列，首先判断是否present bitmap标记了，如果为0，则跳过用null表示，然后在看是否在null bitmap里面标记了，如果为1，表明值为null，最后我们就开始解析真有有数据的列了。

但是，因为我们得到的是一行数据的二进制流，我们怎么知道一列数据如何解析？这里，就要靠TABLE_MAP_EVENT里面的column def以及meta了。

column def定义了该列的数据类型，对于一些特定的类型，譬如MYSQL_TYPE_LONG, MYSQL_TYPE_TINY等，长度都是固定的，所以我们可以直接读取对应的长度数据得到实际的值。但是对于一些类型，则没有这么简单了。这时候就需要通过meta来辅助计算了。

譬如对于MYSQL_TYPE_BLOB类型，meta为1表明是tiny blob，第一个字节就是blob的长度，2表明的是short blob，前两个字节为blob的长度等，而对于MYSQL_TYPE_VARCHAR类型，meta则存储的是string长度。这里，笔者并没有列出MYSQL_TYPE_NEWDECIMAL，MYSQL_TYPE_TIME2等，因为它们的实现实在是过于复杂，笔者几乎对照着MySQL的源码实现的。

搞定了这些，我们终于可以完整的解析一个ROWS_EVENT了，顺带说一下，[python-mysql-replication][4]里面minimal/noblob row image的支持，也是笔者提交的[pull request][5]，貌似是笔者第一次给其他开源项目做贡献。

## 总结

实现MySQL replication protocol的解析真心是一件很有挑战的事情，虽然辛苦，但是让笔者更加深入的学习了MySQL的源码，为后续笔者改进[LedisDB][5]的replication以及更深入的了解MySQL的replication打下了坚实的基础。

话说，现在成果已经显现，不然[go-mysql-elasticsearch][3]不可能如此快速实现，后续笔者准备基于此做一个更新cache的服务，这样我们的代码里面就不会到处出现更新cache的代码了。

[1]: https://github.com/siddontang/go-mysql  "A MySQL toolset"
[2]: https://github.com/siddontang/mixer  "A MySQL Proxy"
[3]: https://github.com/siddontang/go-mysql-elasticsearch  "Sync MySQL into Elasticsearch"
[4]: https://github.com/noplay/python-mysql-replication "Pure python for MySQL replication protocol"
[5]: https://github.com/noplay/python-mysql-replication/pull/103 "Add binlog row minimal and noblob image support"
[6]: https://github.com/siddontang/ledisdb  "A fast NoSQL"