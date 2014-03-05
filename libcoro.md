# 起因

在第一个版本的[libtnet](https://github.com/siddontang/libtnet)开发完成之后，我一直在思考如何让异步方式的网络编程更加简单。

虽然[libtnet](https://github.com/siddontang/libtnet)通过c++ shared_ptr以及function等技术很大程度上面解决了异步代码编写的一些问题，但是仍然会出现代码逻辑被强制拆分的情况。而这个则是项目中童鞋无法很好的使用其进行开发的原因。

所以我考虑让[libtnet](https://github.com/siddontang/libtnet)支持coroutine。

# Coroutine

第一次接触coroutine的概念是在lua里面，记得当时想了很久才算弄明白了coroutine的使用以及原理。在lua中，coroutine的使用如下：

    co = coroutine.create(function ()
            print("begin yield")
            coroutine.yield()
            print("after yield")
        end)
        
    coroutine.resume(co)
    print("after resume")
    
    coroutine.resume(co)
    
我们可以通过resume执行一个新创建或者已经被挂起的coroutine，通过yield挂起当前的coroutine，这样就可以实现类似多线程方式下面的多任务调度。

至于coroutine的原理，很多地方都有说明，主要就在于每个coroutine都有自己的堆栈，这样当coroutine挂起的时候，它的当前执行状态会被完整保留，下次resume的时候就可以接着执行了。

而使用coroutine的好处，我觉得最大的一点在于它将拆分的异步逻辑同步化了，更利于代码编写。

在使用python tornado的时候，我们开始阶段写了太多的callback回调，以至于代码的维护非常困难，而这个则在引入[greenlet](https://github.com/python-greenlet/greenlet)后有了明显好转。

而后续在使用go语言中，因为它原生的支持coroutine（其实在go里面更准确的说法应该是goroutine），写代码非常的方便，所以现在go已经成为了我服务器的首选开发语言，我也用它开发了多个项目（如[mixer](https://github.com/siddontang/mixer)，一个mysql proxy），并且已经在公司项目中实施。

当然，使用coroutine并不是毫无缺点的：

- 每个coroutine都需要维护自己的堆栈，当我们需要创建数以百万计的coroutine的时候，内存的开销就需要考虑了。
- coroutine的切换，都需要保留当前的上下文环境，以便于下次resume的时候接着执行，如果coroutine切换频繁，开销也不小。

# libcoro

很早之前使用luajit的时候，我就知道可以在c++中实现coroutine的功能，在linux中，这通过makecontext，swapcontext等相关函数实现。虽然也可以通过setjmp/longjmp这两个古老的函数实现，但看了luajit的coco就知道，即使在linux下面，它也需要写很多define宏去适配。

所以，我只考虑使用makecontext这套函数族来实现coroutine。虽然swapcontext会有性能问题，详见[这里](http://rethinkdb.com/blog/making-coroutines-fast/)，但早期我还不打算对其进行性能优化。

[libcoro](https://github.com/siddontang/libcoro)是一个简单的c++ coroutine库，只支持linux（因为我们的服务器只有linux的）。

在接口上面，[libcoro](https://github.com/siddontang/libcoro)参考的是lua的coroutine的接口设计，使用非常简单:

    void func1()
    {
        coroutine.yield();
    }
    
    void func2(Coro_t co1)
    {
        coroutine.resume(co1);    
        coroutine.yield();
    }
    
    void func()
    {
        Coro_t co1 = coroutine.create(std::bind(&func1));    
        coroutine.resume(co1);    
        Coro_t co2 = coroutine.create(std::bind(&func2, co1));
        coroutine.resume(co2);
        coroutine.resume(co2);
    }
    
    int main()
    {    
        Coro_t co = coroutine.create(std::bind(&func));
        coroutine.resume(co);
        return 0;
    }

- coroutine.create创建一个coroutine，参数为一个std::function，这样我们就可以通过std::bind非常方便的实现函数闭包了。
- coroutine.resume唤醒一个挂起或者新建的coroutine。
- coroutine.yield挂起当前coroutine。
- coroutine.running获取当前运行的coroutine，如果是主线程调用，则返回0。
- coroutine.status获取coroutine的状态。

后续我考虑将[libtnet](https://github.com/siddontang/libtnet)支持coroutine，不过这可能会成为一个新的网络库了。
