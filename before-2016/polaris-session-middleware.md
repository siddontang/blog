# 起因

[polaris](https://github.com/siddontang/polaris)虽然是模仿tornado开发，但我觉得作为一个go的web框架，还需要提供一些额外的扩展支持。

[polaris](https://github.com/siddontang/polaris)现在已经支持session以及middleware，主要参加django。

[polaris](https://github.com/siddontang/polaris)对于这些额外功能的支持，采取的是注册 + json配置驱动的方式。这个跟go的database/sql有点类似，任何模块都提供一套类似如下的接口:

    type Obj interface {
    
    }

    type Driver interface{
        Open(jsonConfig json.RawMessage) (Obj, error)
    }

    func Register(name string, driver Driver) error
    func Open(name string, jsonConfig json.RawMessage) (Obj, error)

如果我们需要自定义功能，只需要实现自己的driver以及对应的obj，然后Register进去，后续就可以通过Open直接使用了。

对于每个模块的配置，因为[polaris](https://github.com/siddontang/polaris)的整体配置是json，所以我也强制要求参数是json格式的，也就是json.RawMessage，各个模块自行进行Unmarshal处理。

# session

对于一个session对象，无非就是Set，Get，Delete等，[polaris](https://github.com/siddontang/polaris)需要关心的是这个session对应的store。store可以理解为该session的持久化保存位置，可以是db，redis，cookie或者memory。

[polaris](https://github.com/siddontang/polaris)提供的store接口如下：

    type Store interface {
        //get a session by id
        //if no session exist, regenerate another id to new a session
        Get(id string) (*Session, error)
        
        //delete session from store
        Delete(*Session) error
        
        //Save session to stroe
        Save(*Session) error
    }
    
    type Driver interface {
        Open(jsonConfig json.RawMessage) (Store, error)
    }

现阶段，只提供了redis的支持，这里特别说明一下，我是在现在才知道redis有一个setex命令，想想以前经常用set + expire来设置一个key以及超时，想想都汗颜。

对于session的持久化，[polaris](https://github.com/siddontang/polaris)提供了codec的接口，外部可以注册自己的序列化方式，同时在相应的store里面实现。对于一个codec，接口如下：

    //codec for session encode and decode
    type Codec interface {
        Encode(values map[interface{}]interface{}) ([]byte, error)
        Decode(buf []byte) (map[interface{}]interface{}, error)
    }
    
    func RegisterCodec(name string, codec Codec) error
    func GetCodec(name string) (Codec, error)


现阶段，[polaris](https://github.com/siddontang/polaris)提供了gob方式的codec。外部通过GetCodec("gob")就可以获取到。

# middleware

[polaris](https://github.com/siddontang/polaris)的middleware主要提供如下接口：

    type Middleware interface {
        ProcessRequest(env *context.Env) error
        ProcessResponse(env *context.Env) error
    }

context.Env是该次请求的上下文环境，对于每次http请求，[polaris](https://github.com/siddontang/polaris)会首先调用middleware的ProcessRequest，在处理实际对应的restful接口，然后再调用ProcessResponse。

如果Process的时候，返回error，或者env已经finished，[polaris](https://github.com/siddontang/polaris)会终止后续的process操作。这套处理流程是否合适后续在好好考量。

现阶段，[polaris](https://github.com/siddontang/polaris)提供了session middleware的支持。

# Todo

[polaris](https://github.com/siddontang/polaris)采用注册 + json配置的方式，我觉得可以很好的处理后续模块功能的添加问题，后续可以参考django等框架继续完善。
