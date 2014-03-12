# 介绍

[polaris](https://github.com/siddontang/polaris)是一个用go实现的支持restful的web框架，主要参考tornado进行设计。

虽然在go里面搭建一个http server非常的简单，这里强烈推荐[gorilla](http://www.gorillatoolkit.org/)，但并没有很好的对restful模型进行支持。考察了很多开源实现，我决定还是重新造一个轮子，毕竟难度也不怎么大，而且能够根据项目的使用慢慢调整完善。

# 使用

设计[polaris](https://github.com/siddontang/polaris)还是参考了tornado，虽然一段时间不用python了，但是我还是对tornado念念不忘。

在[polaris](https://github.com/siddontang/polaris)，我现在只支持get，post，put，head和delete这几种method，剩下的貌似我也没用到过。只要一个对象，它提供了以上一个或多个接口，就能注册进[polaris](https://github.com/siddontang/polaris)的router中。

先上一个例子：

    type Handler1 struct {

    }

    func (h *Handler1) Prepare(env *Env) {
        fmt.Println("hello prepare")
    }

    func (h *Handler1) Get(env *Env) {
        env.WriteString("hello get")
    }

    func (h *Handler1) Post(env *Env) {
        env.WriteString("hello post")
    }

    type Handler2 struct {

    }

    //id is a captured submatch for regexp url below
    func (h *Handler2) Get(env *Env, id string) {
        env.WriteString("hello " + id)
    }

    r = NewRouter()

    r.Handle("/handler1", new(Handler1))
    r.Handle("/handler2/([0-9]+)", new(Handler2))

    http.Handle("/", r)
    http.ListenAndServe("127.0.0.1:11181", nil)
    
上面，我们实现了两个handler，对于每一个handler，需要实现以下一个或多个接口:

    Get(env *Env)
    Post(env *Env)
    Head(env *Env)
    Put(env *Env)
    Delete(env *Env)
    
Env用以表明该次请求的上下文环境，主要是为了纪念一个内部库stars，以此命名。

鉴于tornado也提供了prepare支持，所以，如果handler提供了Prepare(env *Env)接口，每次调用相应的函数的时候，会优先调用Prepare。

在handler2中，我们可以看到，Get函数多了一个参数id，这主要是为了处理对应的正则url里面含有group的情况。上面handler2对应的url里面有一个([0-9]+) group，所以当url匹配成功之后，我们会将该group实际的数据传递给handler2。

因为正则匹配的结果都是string，所以Get后面的参数都必须是string类型的。实际的类型转换需要handler自行处理。

# 路由规则

[polaris](https://github.com/siddontang/polaris)的路由规则比较简单，参考nginx分为literal pattern和regexp pattern两种。当有请求需要处理的时候，首先查看是否在literal pattern中，如果不是，则根据注册的regexp pattern依次匹配。

上面的例子中，**/handler1**就是literal pattern，而**/handler2/([0-9]+)**则是regexp pattern。在go中，判断一个字符串是否是正则表达式，只要通过regexp.QuoteMeta就可以了，如果得到的值跟原字符串一样，该字符串就是literal pattern。

# 实现

[polaris](https://github.com/siddontang/polaris)核心实现是基于reflect，这里列举一些：

## 判断一个handler是否具有相关接口

- reflect.ValueOf(handler)得到handler的value v
- v.MethodByName("Get")得到get function的value m
- m.Kind() == reflect.Func判断是否存在Get函数

## 判断接口是否满足规范

- m.NumIn()获取该函数输入参数的个数
- m.In(0).Kind() == reflect.Ptr和m.In(0).String() == "*polaris.Env"判断第一个参数必须为\*Env
- m.In(i).Kind() == reflect.String判断第i(i>=1)个参数必须为string

## 根据传入的参数调用相应的函数

- 根据传入的http request method找到对应的函数 m
- values = make([]relfect.Value, nArgs)
- 通过reflect.ValueOf填充参数，第一个参数为\*Env
- m.Call(values)，使用Call进行函数调用

# 后续

现在的[polaris](https://github.com/siddontang/polaris)只支持基本的restful模型，后续我考虑将自己开发的其他库，譬如log，mysql等进行整合，使其真正成为一个可用的restful web framework。

代码在这里[https://github.com/siddontang/polaris](https://github.com/siddontang/polaris)，欢迎大家反馈。