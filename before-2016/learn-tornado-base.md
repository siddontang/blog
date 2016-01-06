# 前言

在python里面，有许多[web framework](http://wiki.python.org/moin/WebFrameworks)。对于我来说，因为很长一段时间都在使用tornado，所以有了一些心得体会。

在这里，要说明一下，tornado采用的是**2.4**版本。

# 架构

tornado是一个典型的prefork + io event loop的web server架构，![Alt text](https://raw.github.com/siddontang/blog/master/asserts/tornado-architecture.png "architecture")

从图上可以看出，tornado的架构是很简单清晰的。

- ioloop是tornado的核心，它就是一个io event loop，底层封装了select，epoll和kqueue，并根据不同的平台选择不同的实现。
- iostream封装了non-blocking socket，用它来进行实际socket的数据读写。
- TCPServer则是通过封装ioloop实现了一个简易的server，同时我们也在这里进行prefork的处理
- HTTPServer则是继承TCPServer实现了一个能够处理http协议的server。
- Application则是实际处理http请求的模块，HTTPServer收到http请求并解析之后会通过Application进行处理。
- RequestHandler和WebSocketHandler则是注册给Application用来处理对应url的。
- WSGIApplication则是tornado用于支持WSGI标准的接口，通过WSGIContainer包装共HTTPServer使用。

# 例子

通过上面的分析，直到tornado的架构是很简单明了的，所以自然我们也能够通过简短的一些代码就能搭建起自己的http server。以一个hello world开始：

    import tornado.web 
    import tornado.httpserver 
    import tornado.ioloop 

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write('Hello World')

    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(8080)
    tornado.ioloop.IOLoop.instance().start()

流程很简单，如下：

- 定义了一个MainHandler，该handler用来处理对应url
- 生成一个Application实例，并设置url dispatch规则，(r"/", MainHandler)就是一个规则，第一个pattern用来表明需要处理的url，内部会使用正则匹配，第二个就是对应url处理的handler
- 生成一个HTTPServer实例，使用Application进行构造，这样HTTPServer处理的http请求就会转给application处理。
- HTTPServer监听一个端口8080，该listen socket会加入ioloop中，用于监听连接的建立。
- ioloop启动，程序进入io event loop模式。

当ioloop start之后，服务器就启动了，后续就是一个http server最基本的流程处理了。

# ReuqestHandler

## pattern and handler

从上面例子可以看出，搭建一个http server很简单，所以我们重点只需要考虑的是如何处理不同的url http请求，这也就是RequestHandler需要做的事情。

我们在创建Application的时候，会指定不同的url pattern需要处理的handler。如下：

    import tornado.web 
    import tornado.httpserver 
    import tornado.ioloop 

    class Index1Handler(tornado.web.RequestHandler):
        def get(self):
            self.write('Index1')

    class Index2Handler(tornado.web.RequestHandler):
        def get(self, data):
            self.write('Index2')
            self.write(data)

    application = tornado.web.Application([
        (r"/index1", Index1Handler),
        (r"/index2/(\w+)", Index2Handler),
    ])

    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(8080)
    tornado.ioloop.IOLoop.instance().start()

在上面的例子中，我们有两个handler，分别处理url path为index1和index2的情况，对于index2来说，我们看到，它后面还需要匹配一个单词。我们通过curl访问如下：

    $ curl http://127.0.0.1:8080/index1
    index1

    $ curl http://127.0.0.1:8080/index2/abc
    index2abc

## http method

RequestHandler支持任何http mthod，包括get，post，head和delete，也就是说，tornado天生支持restful编程模型。

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            pass

        def post(self):
            pass

        def head(self):
            pass

        def delete(self):
            pass

从上面可以看到，我们只需要在handler里面实现自己的get，post，head和delete函数就可以了，这点再次说明tornado的简洁与强大。

# 后续next

这里，只是简单了介绍了一下tornado，后续将会从template，asynchronous，security等分别介绍一下。希望通过这个能让自己对tornado的理解更加深刻，同时也为后续使用其他python web framework做参考。