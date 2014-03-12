# 介绍

[mixer](https://github.com/siddontang/mixer)是一个用go实现的mysql proxy，支持基本的mysql代理功能。

mysql的中间件很多，对于市面上面现有的功能强大的proxy，我主要考察了如下几个：

- mysql-proxy，mysql官方的代理，使用起来并不友好，需要进行lua定制，而且本人对其稳定性和性能存疑。
- Cobar，阿里的东西，品质没的说，但对于我们项目，有点杀鸡用牛刀的感觉，另外我们都不会java。
- Atlas，360出品的基于mysql-proxy的增强版，几乎用c重写了核心框架，性能和稳定性都没话说。

当然，还有很多强大的proxy，我不可能一一涉及，而现阶段我们项目中使用的是Atlas（这算不算给Atlas打了一个广告？）。

既然有这么多的proxy，为什么我还想自己实现一个呢？可能最主要的原因在于兴趣使然吧。

# mysql功能支持

当开始着手进行[mixer](https://github.com/siddontang/mixer)开发的时候，我就知道，[mixer](https://github.com/siddontang/mixer)不是mysql，它不可能proxy所有mysql的功能。所以，我决定[mixer](https://github.com/siddontang/mixer)只支持如下mysql命令：

- COM_QUERY
    + select, insert, update, delete, replace
    + set autocommit
    + set names
    + begin, commit, rollback
- COM_PING
- COM_INIT_DB
- COM_STMT_PREPARE, COM_STMT_EXEC等COM_STMT_*命令，仅支持上述COM_QUERY命令的prepare

[mixer](https://github.com/siddontang/mixer不支持命令挺多的，列举一些：

- set variable。如果支持，[mixer](https://github.com/siddontang/mixer)需要维护每一个变量的状态，增加了复杂度。但[mixer](https://github.com/siddontang/mixer)支持autocommit和names的设置。
- sql text模式的prepare statement。
- show命令。
- 存储过程。

虽然很多功能现阶段没有，但不排除后续支持。

# 高可用方案

[mixer](https://github.com/siddontang/mixer)提供了一套mysql高可用使用方案，现阶段主要功能如下：

- 读写分离，将select发送到slave，其余发送到master执行，事物所有在master执行。现阶段只支持一主一备。
- 主备自动切换，当主mysql不可用，根据相关规则切换到backup mysql执行。

# Todo

[mixer](https://github.com/siddontang/mixer)还不完善，很多功能需要实现，后续优先需要实现的功能：

- parser，将sql进行语法解析，构建AST，在proxy层面就防止一些mysql隐患，譬如注入攻击，delete没有where等。
- 自定义路由，根据路由规则将sql路由到不同mysql执行。譬如根据主键将select语句hash到不同的slave上面执行。
- 统计功能。

代码在这里[https://github.com/siddontang/mixer](https://github.com/siddontang/mixer)。非常希望对proxy感兴趣的童鞋参与进来，共同完善[mixer](https://github.com/siddontang/mixer)，使其成为另一个mysql中间件解决方案。

