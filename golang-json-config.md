最近在用go重构，在先前的代码中，我们使用的ini文件进行配置，但是因为很多历史遗留问题，导致配置混乱，维护困难，自然也需要考虑重构了。

# 通用配置格式

通用的配置格式有很多，常用的就有ini，json，yaml，xml等，当然为了通用我们不考虑自定义的配置格式。那如何选择呢？

首先，xml我们就不用考虑了，到现在为止我都没觉得用这玩意配置起来有多方便，反而很臃肿，可能java系的童鞋会比较青睐。

再来考虑ini，ini文件对于简单应用的配置可以说是非常方便的，如果配置没有太多的层次结构，使用ini就能完全满足我们的需要，即使有，我们也能够通过加入特定前缀来解决。譬如，我们可能有如下redis配置：

    [ModuleA]

    persistent_redis_addr = 127.0.0.1:6379
    persistent_redis_password = admin
    
    cache_redis_addr = 127.0.0.1:6380
    cache_redis_password = admin
    
然后，需求变化了，我们需要另一个持久化redis服务来做相关事情，于是配置就可能变成了下面这样：

    [ModuleA]

    persistent_redis_addr = 127.0.0.1:6379
    persistent_redis_password = admin

    persistent2_redis_addr = 127.0.0.1:6379
    persistent2_redis_password = admin

    cache_redis_addr = 127.0.0.1:6380
    cache_redis_password = admin
    
虽然通过前缀命名能解决层级问题，但总觉得是对程序员不怎么友好的。

# why json

剩下的就是json和yaml，可以这么说，这两个都算是比较好的轻量级配置格式，而且对程序员非常友好，而且go里面都可以通过定义struct，将层级结构的配置映射到相应的struct里面。

但是我还是决定选择json作为我们go代码的默认配置格式，最主要的原因在于go的json包有一个杀手级别的RawMessage实现。而这个在我能找到的yaml包中没有。

RawMessage主要是用来告诉go延迟解析用的。当我们定义了某一个字段为RawMessage之后，go就不会解析这段json，这样我们就可以将其推后到各自的子模块单独解析。

假设有一个功能，后台存储可能是redis或者mysql，但是只会使用一个，可能我们会按照如下方式写配置：

    redis_store : {
        addr : 127.0.0.1
        db : 0
    },

    mysql_store : {
        addr : 127.0.0.1
        db : test
        password : admin
        user : root
    }

    store : redis

对应的class为

    type Config struct {
        RedisStore struct {
            Addr string
            DB int
        }

        MysqlStore Struct {
            Addr string
            DB string
            Password string
            User string
        }

        Store string
    }

如果这时候我们在增加了一种新的store，我们需要在Config文件里面在增加一个新的field，但是实际我们只会使用一种store，并不需要写这么多的配置。

我们可以使用RawMessage来处理：

    type Config struct {
        Store string
        StoreConfig json.RawMessage
    }

如果使用redis，对应的配置文件为

    store: redis
    store_config: {
        addr : 127.0.0.1
        db : 0
    }

如果使用mysql，对应的配置文件为

    store: mysql
    store_config: {
        addr : 127.0.0.1
        db : test
        password : admin
        user : root
    }

go读取配置文件之后，并不会处理RawMessage对应的东西，而是由我们代码自己对应的store模块去处理。这样无论配置文件怎么变动，store模块做了什么变动，都不会影响Config类。

而在各个模块中，我们只需要自己定义相关config，然后可以将RawMessage直接解析映射到该config上面，譬如，对于redis，我们在模块中有如下定义

    type RedisConfig config {
        Addr string `json:"addr"`
        DB int `json:"db"`
    }
    
    func NewConfig(m json.RawMessage) *RedisConfig {
        c := new(RedisConfig)
        
        json.Unmarshal(m, c)
        
        return c
    }
    
# json的不足

使用json也还有很多蛋疼的地方，最大的问题就在于注释，在json中，可不能这样写：

    {
        //this is a comment
        /*this is a comment*/ 
    }
 
但是，我们又不可能不写一点注释来说明配置项是干啥的，所以，通常采用的是引入一个comment字段的方式，譬如：

    {
        "_comment" : "this is a comment",
        "key" : "value"
    }
    
另外，json还需要注意的就是写的时候最后一项不能加上逗号，这样的json会因为格式错误无法解析的。

    {
        "key" : "value",
    }

最后那个逗号可是不能要的，但是实际写配置的时候我们可是经常性的随手加上了。

但是，总的来说，json对于go的项目还是很友好的，我不光在项目中推行了，在自己的开源项目中，也大量的采用了json作为主要的配置文件。