# why asynchronous

tornado是一个异步web framework，说是异步，是因为tornado server与client的网络交互是异步的，底层基于io event loop。但是如果client请求server处理的handler里面有一个阻塞的耗时操作，那么整体的server性能就会下降。

    def MainHandler(tornado.web.RequestHandler):
        def get(self):
            client = tornado.httpclient.HttpClient()
            response = client.fetch("http://www.google.com/")
            self.write('Hello World')

在上面的例子中，tornado server的整体性能依赖于访问google的时间，如果访问google的时间比较长，就会导致整体server的阻塞。所以，为了提升整体的server性能，我们需要一套机制，使得handler处理都能够通过异步的方式实现。

幸运的是，tornado提供了一套异步机制，方便我们实现自己的异步操作。当handler处理需要进行其余的网络操作的时候，tornado提供了一个async http client用来支持异步。

    def MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            client = tornado.httpclient.AsyncHTTPClient()
            def callback(response):
                self.write("Hello World")
                self.finish()

            client.fetch("http://www.google.com/", callback)

上面的例子，主要有几个变化：

- 使用asynchronous decorator，它主要设置_auto_finish为false，这样handler的get函数返回的时候tornado就不会关闭与client的连接。
- 使用AsyncHttpClient，fetch的时候提供callback函数，这样当fetch http请求完成的时候才会去调用callback，而不会阻塞。
- callback调用完成之后通过finish结束与client的连接。

# asynchronous flaw

异步操作是一个很强大的操作，但是它也有一些缺陷。最主要的问题就是在于callback导致了代码逻辑的拆分。对于程序员来说，同步顺序的想法是一个很自然的习惯，但是异步打破了这种顺序性，导致代码编写的困难。这点，对于写nodejs的童鞋来说，可能深有体会，如果所有的操作都是异步，那么最终我们的代码可能写成这样:

    def MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            client = tornado.httpclient.AsyncHTTPClient()
            def callback1(response):

                def callback2(response):
                    self.write("Hello World")
                    self.finish()
                client.fetch("http://www.google.com", callback2)

            client.fetch("http://www.google.com/", callback1)

也就是说，我们可能会写出callback嵌套callback的情况，这个极大的会影响代码的阅读与流程的实现。

# synchronous

我个人认为，异步拆散了代码流程这个问题不大，毕竟如果一个逻辑需要过多的嵌套callback来实现的话，那么我们就需要考虑这个逻辑是否合理了，所以异步一般也不会有过多的嵌套层次。

虽然我认为异步的callback问题不大，但是如果仍然能够有一套机制，使得异步能够顺序化，那么对于代码逻辑的编写来说，会方便很多。tornado有一些机制来实现。

## yield

在python里面如果一个函数内部实现了yield，那么这个函数就不是函数了，而是一个生成器，它的整个运行机制也跟普通函数不一样，举一个例子:

    def test_yield():
        print 'yield 1'
        a = yield 'yielded'
        print 'over', a

    t = test_yield()
    print 'main', type(t)
    ret = t.send(None)
    print ret
    try:
        t.send('hello yield')
    except StopIteration:
        print 'yield over'

输出结果如下：
    
    main <type 'generator'>
    yield 1
    yielded
    over hello yield
    yield over

从上面可以看到，test_yield是一个生成器，当它第一次调用的时候，只是生成了一个Generator，不会执行。当第一次调用send的时候，生成器被resume，开始执行，然后碰到yield，就挂起，等待下一次被send唤醒。当生成器执行完毕，会抛出StopIteration异常，供外部send的地方知晓。

因为yield很方便的提供了一套函数挂起，运行的机制，所以我们能够通过yield来将原本是异步的流程变成同步的。

## gen

tornado有一个gen模块，提供了Task和Callback/Wait机制用来支持同步模型，以task为例：

    def MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        @tornado.gen.engine
        def get(self):
            client = tornado.httpclient.AsyncHTTPClient()
            response = yield tornado.gen.Task(client.fetch, "http://www.google.com/")
            self.write("Hello World")
            self.finish()

可以看到，tornado的gen模块就是通过yield来进行同步化的。主要有如下需要注意的地方：

- 使用gen.engine的decorator，该函数主要就是用来管理generator的流程控制。
- 使用了gen.Task，在gen.Task内部，会生成一个callback函数，传给async fetch，并执行fetch，因为fetch是一个异步操作，所以会很快返回。
- 在gen.Task返回之后使用yield，挂起
- 当fetch的callback执行之后，唤醒挂起的流程继续执行。

可以看到，使用gen和yield之后，原先的异步逻辑变成了同步流程，在代码的阅读性上面就有不错的提升，不过对于不熟悉yield的童鞋来说，开始反而会很迷惑，不过只要理解了yield，那就很容易了。

## greenlet

虽然yield很强大，但是它只能挂起当前函数，而无法挂起整个堆栈，这个怎么说呢，譬如我想实现下面的功能:

    def a():
        yield 1

    def b():
        a()

    t = b()
    t.send(None)

这个通过yield是无法实现的，也就是说，a里面使用yield，它是一个生成器，但是a的挂起无法将b也同时挂起。也就是说，我们需要一套机制，使得堆栈在任何地方都能够被挂起和恢复，能方便的进行栈切换，而这套机制就是coroutine。

最开始使用coroutine是在lua里面，它原生提供了coroutine的支持。然后在使用luajit的时候，发现内部是基于fiber(win)和context(unix)，也就是说，不光lua，其实c/c++我们也能实现coroutine。现在研究了go，也是内置coroutine，并且这里极力推荐一篇[slide](http://concur.rspace.googlecode.com/hg/talk/concur.html#table-of-contents)。

python没有原生提供coroutine，不知道以后会不会有。但有一个greenlet，能帮我们实现coroutine机制。而且还有人专门写好了tornado与greenlet结合的模块，叫做[greenlet_tornado](https://github.com/mopub/greenlet-tornado)，使用也很简单

    class MainHandler(tornado.web.RequestHandler):
        @greenlet_asynchronous
        def get(self):
            response = greenlet_fetch('http://www.google.com')
            self.write("Hello World")
            self.finish()

可以看到，使用greenlet，能更方便的实现代码逻辑，这点比使用gen更方便，因为这些连写代码的童鞋都不用去纠结yield问题了。

# 总结

这里只是简单的介绍了tornado的一些异步处理流程，以及将异步同步化的一些方法。另外，这里举得例子都是网络http请求方面的，但是server处理请求的时候，可能还需要进行数据库，本地文件的操作，而这些也是同步阻塞耗时操作，同样可以通过异步来解决的，这里就不详细说明了。

