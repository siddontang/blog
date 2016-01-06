# HTTP

libtnet提供了简单的http支持，使用也很简单。

一个简单的http server：

    void onHandler(const HttpConnectionPtr_t& conn, const HttpRequest& request)
    {
        HttpResponse resp;
        resp.statusCode = 200;
        resp.setContentType("text/html");
        resp.body.append("Hello World");    
        conn->send(resp);
    }

    TcpServer s;
    HttpServer httpd(&s);
    httpd.setHttpCallback("/test", std::bind(&onHandler, _1, _2));
    httpd.listen(Address(80));
    s.start(4);
    
我们对http server的**"/test"**注册了一个handler，当用户访问该url的时候，就会显示"Hello World"。

同样，http client的使用也很简单：
    
    void onResponse(IOLoop* loop, const HttpResponse& resp)
    {
        cout << resp.body << endl;
        loop->stop();
    }

    IOLoop loop;
    HttpClientPtr_t client = std::make_shared<HttpClient>(&loop);
    client->request("http://127.0.0.1:80/test", std::bind(&onResponse, &loop, _1));
    loop.start(); 
    
这里，我们使用了一个http client，向server请求"/test"的内容，当服务器有响应之后，会调用响应的回调函数。

# HTTP Parser

对于http的解析，我采用的是[http parser](https://github.com/joyent/http-parser)，因为它采用的是流式解析，同时非常容易集成进libtnet。

使用http parser只需要设置相应的回调函数即可。http parser有如下几种回调：

- message begin，解析开始的时候调用
- url，解析url的时候调用
- status complete，http response解析status的时候调用
- header field，解析http header的field调用
- header value，解析http header的value调用
- headers complete，解析完成http header调用
- body，解析http body调用
- message complete，解析完成调用

这里特别需要注意的是http header的解析，因为http parser将其拆分成了两种回调，所以我们在处理的时候需要记录上一次header callback是field的还是value的。在解析field的时候，如果上一次是value callback，那我们就需要将上一次解析的field和value保存下来，而该次的解析则是一个新的field了。

另外，http parser还提供了upgrade的支持，所以我们很方便的就能区分该次请求是否为websocket。

# Websocket

libtnet也提供了websocket的支持，现阶段，只支持[RFC6455](http://tools.ietf.org/html/rfc6455)。

当libtnet通过http parser发现该次请求为websocket的时候，就进入了websocket的流程。websocket的使用也很简单，当握手成功之后，后续的所有通讯就是纯粹的tcp通信了。

一个简单的websocket server：

    void onWsCallback(const WsConnectionPtr_t& conn, WsEvent event, const void* context)
    {
        switch(event)
        {
            case Ws_CloseEvent:
                break;
            case Ws_MessageEvent:
                {
                    const string& str = *(const string*)context;
                    conn->send(str);
                }
                break;
            case Ws_PongEvent:
                break;
            default:
                break;
        }
    }

    TcpServer s;
    HttpServer httpd(&s);
    httpd.setWsCallback("/push/ws", std::bind(&onWsCallback, _1, _2, _3));    
    httpd.listen(Address(80));
    s.start();
    
可以看到，websocket的callback机制也类似于libtnet connection的callback机制，用户需通过event + context的方式来处理该次回调的数据。

libtnet对websocket的frame的处理参照的是tornado的websocket模块。也能够组合多frame的数据，外部只需要关注Ws_MessageEvent即可。

websocket client的实现也很简单：

    void onWsConnEvent(const WsConnectionPtr_t& conn, WsEvent event, const void* context)
    {
        switch(event)
        {
            case Ws_OpenEvent:
                conn->send("Hello world");
                break;    
            case Ws_MessageEvent:
                {
                    const string& msg = *(const string*)context;
                    
                    LOG_INFO("message %s", msg.c_str());
                    conn->close();                        
                }
                break;
            default:
                break;
        }    
    }

    IOLoop loop;
    WsClientPtr_t client = std::make_shared<WsClient>(&loop);    
    client->connect("ws://127.0.0.1:80/push/ws", std::bind(&onWsConnEvent, _1, _2, _3));
    loop.start();
