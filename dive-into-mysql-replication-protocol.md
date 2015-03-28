# Dive into MySQL replication protocol 

## Preface 

Let’s consider following scenario, we store huge data in MySQL and want to know data changes immediately(data inserted, deleted or updated), then do something for these changes like updating associated cache data.

Using MySQL trigger + UDF may be a feasible way, but I have not used it and would not recommend this. If we have many tables, maintaining so many triggers is horrible. At the same time, UDF may fail so that we will lost some changes, and even worse, a bad UDF implementation may block service or even crash MySQL.

We need a better solution, and this is binlog. MySQL records every changes into binlog, so if we can sync the binlog, we will know the data changes immediately, then do something. 

Using rsync may be a good way, but I prefer another: acting as a slave and using MySQL replication protocol to sync.

## MySQL Packet 

First, we must know how to send or receive data with MySQL, and this is packet:

Split the data into packets of size 16MB.
Prepend to each chunk a packet header.  

A packet header has 3 bytes for following payload data length and 1 byte for sequence ID. 

The sequence ID is incremented with each packet between client and server communication, it starts at 0 and is reset to 0 when a new command begins, e.g, we send a packet, the sequence ID is 0, and the response packet sequence ID must be 1, if not, some error happens between the communication. 

So a packet looks like below:

```
3              payload length
1              Sequence ID
string[len]    payload
```

As you see, sending data with packet is very easy, I have implemented a base library in [go-mysql][1] packet pkg.

## Connection phase 

When we first connect to MySQL server, we must do a handshake to let server authorizing us to communicate with it later, this is called connection phase. 

Let’s consider  a popular common connection phase.

+ Client connects server using socket connect API.
+ Server sends a initial handshake packet which contains server’s capabilities and a 20 bytes random salt.
+ Client answers with a handshake response packet, telling server its capabilities and a  20 bytes scrambled password.
+ Server response ok packet or err packet. 

Although MySQL supports many authentication method, I only use username + password, you can see codes in [go-mysql][1] client and server pkg.

Now we know how to send/receive data, how to establish the connection, and it’s time to move on for syncing binlog. :-)

## Register a Slave 

When we want to sync binlog, we must register a slave at master first, that is acting as a pseudo slave. 

We only need to send COM_REGISTER_SLAVE command, the payload is:

```
1              COM_REGISTER_SLAVE
4              server-id
1              slaves hostname length
string[$len]   slaves hostname
1              slaves user len
string[$len]   slaves user
1              slaves password len
string[$len]   slaves password
2              slaves mysql-port
4              replication rank
4              master-id
```

We must use an unique server id for our pseudo slave, I prefer using a ID bigger than 1000 which is different from our real MySQL servers. 

## Dump BinLog from master 

After register, we should send COM_BINLOG_DUMP command, it is very easy too, payload is:

```
1              COM_BINLOG_DUMP
4              binlog-pos
2              flags
4              server-id
string[EOF]    binlog-filename
```

If the binlog filename is empty, server will use the first known binlog. 

You can set flags to 1 to tell server to reply a EOF packet instead of blocking connection if there is no more binlog event. But I prefer blocking so that if there is any new event, server can send to us immediately. 

Although MySQL supports COM_BINLOG_DUMP_GTID commands, I still prefer using command binlog filename + position, because it is very easy and for our pseudo slave, we only need to sync binlog, save it in a place and let other applications like MHA use it. 

After we send COM_BINLOG_DUMP command, server will response a binlog network stream which contains many binlog events. 

## BinLog Event 

A MySQL binlog event has many versions, but we only care version 4(MySQL 5.0+) and don’t support earlier versions.

A binlog event contains three parts, header, post header and payload, but I like parsing post header and payload at same time. 

A event header looks below:

```
4              timestamp
1              event type
4              server-id
4              event-size
4              log pos
2              flags
```

You can see all binlog event types here. The log pos is position of the next event, we can save this position for resuming syncing later. 

The first binlog event in the binlog file is FORMAT_DESCRIPTION_EVENT, this event is very important, we use it to determine table ID size for row based event, and check the last 5 bytes to see whether CRC32 is enabled or not for the following events, etc…

Another event we should care is ROTATE_EVENT, it tells us a new binlog coming. 

Parsing a binlog event is very easy, we only need to refer the document and parse it one by one, except row based replication event. 

## Row Based Replication  

Parsing row based replication event is very hard if we want to know the real data changes for a column in one row. 

First, we must parse TABLE_MAP_EVENT, payload is:

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

The post_header_len is saved in FORMAT_DESCRIPTION_EVENT, column count and def tells us how many columns in one row and their data types, like tiny int, short int, float, set, etc. column meta def is tricky, but we must use it to parse following ROWS_EVENT.

A ROWS_EVENT contains WRITE_ROWS_EVENT(insert), DELETE_ROWS_EVENT(delete) and UPDATE_ROWS_EVENT(update), every event contains v0, v1 and v2 three versions, most of time, we only need to care v1 and v2. 

A ROWS_EVENT looks below:

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

I promise that if you don’t dive into the MySQL source, you can not understand how to parse it at all. 

First, let’s see columns-present-bitmap, for a column, it will not be saved in the row data if the associated bit in columns-present-bitmap is 0, and we will skip this column when parsing. 

For every row data, first we should calculate the null-bitmap, a big pitfall here,  we calculate columns-present-bitmap using (num of columns+7)/8, but we must use (bits set in ‘columns-present-bitmap'+7)/8 for null-bitmap. (You should google bits set if you don’t understand it).

From MySQL 5.6, it supports another two binlog row images: minimal and noblob. For minimal row image update, if we have 16 columns and only the first column data changed, if we use (num of columns+7)/8, we should use 2 bytes to store null bitmap, but if we use (bits set in ‘columns-present-bitmap’+7)/8, we will only use 1 bytes to store null bitmap, saving 1 byte(is it really necessary?). By the way, I sent a pull request for python-mysql-replication to handle minimal and noblob row image paring.

Now we get column-present-bitmap and null-bitmap, for a column, if it’s not set in column-present-bitmap or set in null-bitmap, we will know that this column is null for the current row data.

Then we will parse the rest of none null columns. For some special  columns, like MYSQL_TYPE_LONG or MYSQL_TYPE_FLOAT, we can know the data length directly, e.g, MYSQL_TYPE_LONG is 4 bytes and MYSQL_TYPE_TINY_INT is 1 byte.

But for other columns, we should use column meta in TABLE_MAP_EVENT to help us determine the data length. For example, a MYSQL_TYPE_BLOB column, if meta is 1, the data is tiny blob and the first 1 byte in data is the length for payload, if meta is 2, the data is short blob and the first 2 bytes is the lenght for payload. 

Paring the real data is very hard and I can not illustrate here fully and clearly. I hope you can see the source in MySQL or [go-mysql][1] replication pkg if you have some interest. 

## Semi Sync Replication  

At first, I didn’t support semi sync replication in [go-mysql][1], but after I develop [go-mysql-elasticsearch][3], I realize that if I want to sync MySQL changes into elasticsearch more quickly and immediately, supporting semi sync replication is a good choice and it’s easy to do like below: 

+ Check whether master supports semi sync or not, using “SHOW VARIABLES LIKE ‘rpl_semi_sync_master_enabled’”.
+ Tell master we support semi sync using “SET @rpl_semi_sync_slave = 1”.

If all is ok, server will prepend two byte before every binlog event. The first byte is 0xef indicating semi sync and the second byte is semi sync ACK flag, if 1, we must reply a semi sync ACK packet.

It now seems that this is a wise decision. In facebook, they even develop a semi sync binlog, you can see more here. I develop a similar go-mysqlbinlog supporting semi sync too, but it still needs improvement for production environment. 

## Summary 

Learning mysql protocol is a hard but happy journey for me, and I have done some interesting things too, like mixer, a MySQL proxy which the modified version(cm) has been used in production in wandoujia company. [go-mysql-elasticsearch][2], a tool to sync MySQL data into elasticsearch immediately. 

Now I have been developing go-mysql, a powerful MySQL toolset. I would be very glad if some people find it can help them for their own MySQL development too.

Later I will also try to use go-mysqlbinlog + MHA to build our own MySQL HA solution, I don’t know perl, but luckily, I can understand MHA code.

Below is my contact, any advice and feedback is very welcome.

+ Email: siddontang@gmail.com

+ Skype: live:siddontang_1

[1]: https://github.com/siddontang/go-mysql  "A MySQL toolset"
[2]: https://github.com/siddontang/mixer  "A MySQL Proxy"
[3]: https://github.com/siddontang/go-mysql-elasticsearch  "Sync MySQL into Elasticsearch"
