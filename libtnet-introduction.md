libtnet是一个用c++编写的高性能网络库，它在设计上面主要参考tornado，为服务端网络编程提供简洁而高效的接口，非常易于使用。

# Echo Server

    void onConnEvent(const ConnectionPtr_t& conn, ConnEvent event, const void* context)
    {
        switch(event)
        {
            case Conn_ReadEvent:
                {
                    const StackBuffer* buffer = static_cast<const StackBuffer*>(context);
                    conn->send(string(buffer->buffer, buffer->count));
                }
                break;
            default:
                break;
        }    
    }
    
    int main()
    {
        TcpServer s;
        s.listen(Address(11181), std::bind(&onConnEvent, _1, _2, _3));
    
        s.start();
        
        return 0;
    }
    
当程序启动，服务监听本地11181端口，我们使用telnet测试：

    root@tnet:~# telnet 127.0.0.1 11181
    Trying 127.0.0.1...
    Connected to 127.0.0.1.
    Escape character is '^]'.
    hello world
    hello world
   
可以看到，libtnet在使用上面非常简单，在listen的时候，指定一个回调函数，当有新的连接到来的时候，该回调函数就会与该connection进行绑定，这样该connection的任何事件都能通过回调进行处理。

在上面那个例子中，我们只关心了connection的ReadEvent，也就是读事件，然后将读取到的所有数据原封不动的转发回去。

# Http Server

    void onHandler(const HttpConnectionPtr_t& conn, const HttpRequest& request)
    {
        HttpResponse resp;
        resp.statusCode = 200;
        resp.body.append("Hello World");
    
        conn->send(resp);
    }

    int main()
    {
        TcpServer s;
        HttpServer httpd(&s);   
        httpd.setHttpCallback("/abc", std::bind(&onHandler, _1, _2));
 
        httpd.listen(Address(11181));    
        s.start(4);
    
        return 0;
    } 

当server启动，程序使用本机11181端口提供http服务。我们使用curl测试。

    curl http://127.0.0.1:11181/abc
    
    return: hello world

可以看到，使用http server也非常简单，我们只需要对相应的路径绑定一个callback回调，当有请求发生的时候，对应的callback执行。

使用benchmark测试，发现性能也不错RPS能到16000+，在512MB，单核CPU下面进行ab压测，具体可以参考[benchmark](https://github.com/siddontang/libtnet/wiki/Benchmark)。

# Webscoket Server

    void onWsCallback(const WsConnectionPtr_t& conn, WsEvent event, const void* context)
    {
        switch(event)
        {
            case Ws_MessageEvent:
                {
                    const string& str = *(const string*)context;
                    conn->send("hello " + str);
                }
                break;
            default:
                break;
        }
    }
    
    int main()
    {
        TcpServer s;
        
        HttpServer httpd(&s);
        
        httpd.setWsCallback("/push/ws", std::bind(&onWsCallback, _1, _2, _3));    
    
        httpd.listen(Address(11181));
    
        s.start();
        
        return 0; 
    }

libtnet同样提供了websocket [RFC6455](http://tools.ietf.org/html/rfc6455)的支持，使用方法同http server，只需要对相应的path注册特定的回调，就可以很方便的进行websocket交互。

# Client

libtnet不光提供了server层面的相关功能，同时也集成了**http client**，**websocket client**以及**redis client**。使得libtnet也能方便的进行客户端网络功能的开发。对于具体的使用，可以参考[example](https://github.com/siddontang/libtnet/tree/master/test)。

# 设计上面的考量

libtnet只支持linux版本，虽然做一个跨平台的通用库是一件吸引力非常大的事情，但是综合考虑之后，我决定只做linux版本的，主要有以下几个原因：

- Linux下面使用prefork + epoll是一种非常高效的网络编程模型，性能强悍，实现简单。虽然unix下面有kqueue，windows下面有IOCP，但是没必要为了适配所有得操作系统将代码写的复杂。
- Linux在系统层面上面就提供了很多高性能的函数，譬如timerfd，eventfd等，不光性能提升，同时也简化了很多代码实现。
- Linux在服务器编程领域的使用率很高，专门做精一个平台就够了。

因为高性能的网络编程通常都是使用异步的编程方式，所以经常可以看到代码被异步拆的特别分散，不利于编写。所以我在libtnet里面大量的使用了c++ bind以及shared_ptr技术，用来模拟函数闭包，以及解决对象生命周期管理问题，简化代码的编写。并且我也使用了c++ 0x相关技术，gcc的版本至少要在4.4以上。

对于如何设计以及使用libtnet，后续我会有更加详细的说明。
