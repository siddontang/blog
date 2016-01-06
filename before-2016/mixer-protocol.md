# 综述

要实现一个mysql proxy，首先需要做的就是理解并实现mysql通讯协议。这样才能通过proxy架起client到server之间的桥梁。

[mixer](https://github.com/siddontang/mixer)的mysql协议实现主要参考mysql官方的[internal manual](http://dev.mysql.com/doc/internals/en/client-server-protocol.html)，并用Wireshark同时进行验证。在实现的过程中，当然踩了很多坑，这里记录一下，算是对协议分析的一个总结。

需要注意的是，[mixer](https://github.com/siddontang/mixer)并没有支持所有的mysql协议，譬如备份，存储过程等，主要在于精力有限，同时也为了实现简单。

# 数据类型

mysql协议只有两种基本的数据类型，integer和string。

## integer

integer包括fixed length integer和length encoded integer两种，对于length encoded integer，用的地方比较多，这里详细说明一下。

对于一个integer，我们按照如下的方式将其转成length encoded integer：

+ 如果value < 251，使用1 byte
+ 如果value >= 251 同时 value < 2 \*\* 16，使用fc + 2 byte
+ 如果value >= 2 \*\* 16 同时 value < 2 \*\* 24，使用fd + 3 byte 
+ 如果value >= 2 \*\* 24 同时 value < 2 \*\* 64，使用fe + 8 byte

相应的，对于一个length encoded integer，我们可以通过判断第一个byte的值来转成相应的integer。

## string

string包括：

+ fixed length string，固定长度string
+ null terminated string，以null结尾的string
+ variable length string，通过另一个值决定长度的string
+ length encoded string，通过起始length encoded integer决定长度的string
+ rest of packet string，从当前位置到包结尾的string

# Packet

在mysql中，如果client或server要发送数据，它需要将数据按照(2 \*\* 24 - 1)拆分成packet，给每一个packet添加header，然后再以此发送。

对于一个packet，格式如下：

    3              payload length
    1              sequence id
    string[len]    payload

前面3个字节表明的是该packet的长度，每个packet最大不超过16MB。第4个字节表明的是该packet的序列号，从0开始，对于多个packet依次递增，等到下一个新的命令发送数据的时候才重置为0。前面4个字节组成了一个packet的header，后面就是该packet实际的数据。

因为一个packet最大能发送的数据位16MB，所以如果需要发送大于16MB的数据，就需要拆分成多个packet进行发送。

通常，server会回给client三种类型的packet

- OK Packet，操作成功
- Err Packet，操作失败
- EOF Packet，end of file

# 登陆交互

要实现proxy，首先需要解决的就是登陆问题，包括proxy模拟server处理client的登陆，proxy模拟client登陆server。

为了简单，[mixer](https://github.com/siddontang/mixer)只支持username + password的方式进行登陆，这应该也是最通用的登陆方式。同时不支持ssl以及compression。

一个完整的登陆流程如下：

- client首先connect到server
- server发送initial handshake packet，包括支持的capability，一个用于加密的随机salt等
- client返回handshake结果，包括自己支持的capability，以及用salt加密的密码
- server验证，如果成功，返回ok packet，否则返回err packet并关闭连接

这里，不得不说实现登陆协议的时候踩过的一个很大的坑，因为我使用的是HandshakeV10协议，在文档里面，协议有这样的规定：

    if capabilities & CLIENT_SECURE_CONNECTION {
        string[$len]   auth-plugin-data-part-2 ($len=MAX(13, length of auth-plugin-data - 8))
    }
   
如果根据文档的说明，算出来auth-plugin-data-part-2的长度是13，因为auth-plugin-data的长度是20。但是，实际情况是，auth-plugin-data-part-2的长度应该为12，第13位一直为0。只有这样，我们才能根据salt算出正确的加密密码。这一点，在mysql-proxy官方的文档，以及多个msyql client driver上面，Wireshark的分析中都是如此，在go-sql-driver中，作者都直接写了如下的注释：

	// second part of the password cipher [12? bytes]
	// The documentation is ambiguous about the length.
	// The official Python library uses the fixed length 12
	// which is not documented but seems to work.

可想而知，这个坑有多坑爹。至少我开始是栽在上面了。加密老是不对。

# Command

搞定了登陆，剩下的就是mysql的命令支持，[mixer](https://github.com/siddontang/mixer)只实现了基本的命令。主要集中在text protocol以及prepared statment里面。

## COM_PING

最基本的ping实现，用来检查mysql是否存活。

## COM_INIT_DB

虽然叫init db，其实压根干的事情就跟use db一样，用来切换使用db的。

## COM_QUERY

可以算是最重要的一个命令，我们在命令行使用的多数mysql语句，都是通过该命令发送的。

在COM_QUERY中，[mixer](https://github.com/siddontang/mixer)主要支持了select，update，insert，delete，replace等基本的操作语句，同时支持begin，commit，rollback事物操作，还支持set names和set autocommit。

COM_QUERY有4中返回packet

- OK Packet
- Err Packet
- Local In File（不支持）
- Text Resultset

这里重点说明一下text resultset，因为它包含的就是我们最常用的select的结果集。

一个text resultset，包括如下几个包：

- 一个以length encoded integer编码的column-count packet
- column-count个column定义packet
- eof packet
- 一个或者多个row packet，每个row packet有column-count个数据
- eof packet或者err packet

对于一个row packet的里面的数据，我们通过如下方式获取：

- 如果值为NULL，那么就是0xfb
- 否则，任何值都是用length encoded string表示

## COM_STMT_*

COM_STMT_族协议就是通常的prepared statement，当我在atlas群里面说支持prepared statement的时候，很多人以为我支持的是在COM_QUERY中使用的prepare，execute和deallocate prepare这组语句。其实这两个还是很有区别的。

为什么我不现在不想支持COM_QUERY的prepare，主要在于这种prepare需要进行变量设置，[mixer](https://github.com/siddontang/mixer)在后端跟server是维护的一个连接池，所以对于client设置的变量，proxy维护起来特别麻烦，并且每次跟server使用新的连接的时候，还需要将所有的变量重设，这增大了复杂度。所以我不支持变量的设置，这点看cobar也是如此。既然不支持变量，所以COM_QUERY的prepare我也不会支持了。

COM_STMT_*这组命令，主要用在各个语言的client driver中，所以我觉得只支持这种的prepare就够了。

对于COM_STMT_EXECUTE的返回结果，因为prepare的语句可能是select，所以会返回binary resultset，binary resultset组成跟前面text resultset差不多，唯一需要注意的就是row packet采用的是binary row packet。

对于每一个binary row packet，第一个byte为0，后面紧跟着一个null bitmap，然后才是实际的数据。

在binary row packet中，使用null bitmap来表明该行某一列的数据为NULL。null bitmap长度通过 (column-count + 7 + 2) / 8计算得到，而对于每列数据，如果为NULL，那么它在null bitmap中的位置通过如下方式计算：

    NULL-bitmap-byte = ((field-pos + offset) / 8)
    NULL-bitmap-bit  = ((field-pos + offset) % 8)

offset在binary resultset中为2，field-pos为该列的位置。

对于实际非NULL数据，则是根据每列定义的数据类型来获取，譬如如果type为MYSQL_TYPE_LONGLONG，那么该数据值的长度就是8字节，如果type为MYSQL_TYPE_STRING，那么该数据值就是一个length encoded string。

# 后记

我通过Wireshark分析了一些mysql protocol，主要在[这里](https://github.com/siddontang/mixer/blob/master/doc/protocol.txt)，这里不得不强烈推荐wireshark，它让我在学习mysql protocol过程中事半功倍。

mixer的代码在这里[https://github.com/siddontang/mixer](https://github.com/siddontang/mixer)，欢迎反馈。
