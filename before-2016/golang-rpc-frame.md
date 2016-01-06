# 使用go reflect实现一套简易的rpc框架

## go jsonrpc

在实际项目中，我们经常会碰到服务之间交互的情况，如何方便的与远端服务进行交互，就是一个需要我们考虑的问题。

通常，我们可以采用restful的编程方式，各个服务提供相应的web接口，相互之间通过http方式进行调用。或者采用rpc方式，约定json格式进行数据交互。

在我们的项目中，服务端对用户客户端提供的是restful的接口方式，而在服务器内部，我们则采用rpc方式进行服务之间的交互。

go语言本来就提供了jsonrpc的支持，所以自然开始我们就直接使用jsonrpc。jsonrpc的使用非常简单，对于调用端来说，就如同一个函数调用，如下：

    args := &Args{7, 8}
    reply := new(Reply)
    err := client.Call("Arith.Add", args, reply)
    
上面是go jsonrpc自带的一个例子，可以看到，虽然我们通过call(rpcName, inParams, outParams)这样的形式可以很方便的进行rpc的调用，但是跟go实际的函数调用还是稍微有一点区别，对我来说，这么使用总觉得很别扭。

## 我觉得方便的rpc使用方式

对于go jsonrpc来说，它的调用格式是这样的

    err := call(name, in, out)
    
但是对我来说，我希望采用这样的调用方式：

    out, err := RpcFunc(in)

假设server端有如下的一个RPC函数，注册的rpc name为testrpc。

    func Test(id int) (int, error)
    
对于client端来说，我希望的使用方式是这样:

    var rpcTest func(id int) (int, error)
    MakeRpc("testrpc", &rpcTest)
    
    id, err := rpcTest(10)

在client端，我们首先声明了跟server Test类型完全一致的一个函数变量。然后通过MakeRpc接口将其命名为testrpc，并且将rpcTest绑定到实际的rpc函数上面，最后rpcTest就跟普通函数的调用一样。
   
可以看到，这种使用方式跟jsonrpc最大的不同在于rpc的返回值直接就是函数自身的返回值。而这个可能更符合我使用go函数的习惯。

## 实现

实现一套rpc框架需要考虑server，client以及包协议的问题。

### 包协议

我使用了最简单的包头 + 实际数据的做法，包头使用一个4字节的int表示后续数据的长度。而对于实际的rpc数据，我采用的是gob进行打包解包。

为什么选用gob而不是json？主要在于我不想自己做数据类型的转换，在json中，int类型的encode，decode会变成float类型的，如果函数需要的参数是int，json decode之后还需要我们自己根据参数实际的类型进行转换。增加了复杂度。而gob则在encode时候会加上实际的数据类型，这样decode之后我就能直接使用。

而且gob还支持注册自定义的类型，但是为了简单，建议只支持基本的数据类型，因为对于rpc来说，传递复杂的数据类型进行函数调用，我总觉得有点复杂，这在设计上面已经有问题了。

### server

在server需要解决的问题就是rpc函数注册并通过名字能进行该rpc函数调用。而这个通过reflect就能非常方便的实现，一个通过函数名字进行函数调用的例子：

    func Test(id int) (string, error) {
        return "abc", nil
    }
    
    funcmap  = map[string]reflect.Value{}
    
    v := reflect.ValueOf(Test)
    
    funcmap["test_rpc"] = v
    
    args := []reflect.Value{reflect.ValueOf(10)}

    funcmap["test_rpc"](args)
    
    
### client

在client层，我们需要关注在声明一个rpc原型的函数变量之后，如何将其替换成另一个函数进行rpc调用。我们可以通过reflect的MakeFunc函数方便的做到，go自身的例子：

    swap := func(in []reflect.Value) []reflect.Value {
        return []reflect.Value{in[1], in[0]}
    }
    
	 makeSwap := func(fptr interface{}) {
        fn := reflect.ValueOf(fptr).Elem()
        v := reflect.MakeFunc(fn.Type(), swap)
        fn.Set(v)
    }

    var intSwap func(int, int) (int, int)
    makeSwap(&intSwap)
    fmt.Println(intSwap(0, 1))

MakeFunc的原理在于，根据传入的函数变量的类型，创建一个新的函数，该函数调用的是我们指定的另一个函数。

同时，我们得到传入变量的指针，并用新的函数重新给该变量赋值。

### error处理

因为rpc调用可能会出现其他错误，譬如网络断线，gob encode错误等，client在调用的时候必须得处理这些错误，暴力的作法就是如果是这种内部错误，我们直接panic，但是我觉得太不友好，所以我们约定，所有的rpc函数在最后一个返回值必须是error。这样就是是rpc内部的错误，我们也能够通过error返回。

在注册rpc的时候，我们可以通过判断最后一个返回值是否是interface，同时是否具有Error函数来强制要求必须为error。如下

    v := reflect.ValueOf(rpcFunc)
    
    nOut := v.Type().NumOut()
    
    if nOut == 0 || v.Type().Out(nOut-1).Kind() != reflect.Interface {
        err = fmt.Errorf("%s return final output param must be error interface", name)
        return
    }
    
    _, b := v.Type().Out(nOut - 1).MethodByName("Error")
    if !b {
        err = fmt.Errorf("%s return final output param must be error interface", name)
        return
    }

但是，如果在MakeFunc里面直接返回error，会出现“reflect: function created by MakeFunc using closure returned wrong type: have *errors.errorString for error”这样的问题，主要在于reflect.Value需要知道我们error的接口类型，参考[这里](http://grokbase.com/t/gg/golang-nuts/139qpwv48h/go-nuts-returning-interface-value-from-reflect-makefunc)。

所以，我们通过如下方式对error进行处理，转成相应的reflect.Value

    v := reflect.ValueOf(&e).Elem()

### nil处理

在实际rpc中，我们可能还会面临参数为nil的问题，如果直接对nil进行reflect.ValueOf，是得不到我们期望的类型的，这时候的Kind是0，reflect压根不能将其正确的转换成函数实际的类型。

当碰到nil的情况，我们只需要根据当前函数参数实际的类型，生成一个Zero Value，就可以很方便的解决这个问题：

假设函数第一个返回值为nil，那么我们这样

    v := reflect.Zero(fn.Type().Out(0))

## 代码

最开始写了一个代码片段验证自己的想法，在[这里](https://gist.github.com/siddontang/9088489)。

一个rpc frame的完整[实现](https://github.com/siddontang/golib/tree/master/rpc)。
    