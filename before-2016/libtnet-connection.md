libtnet只支持IPv4 TCP Connection，之所以这么做都是为了使得实现尽可能的简单。我们主要在Connection类中封装了对tcp连接的操作。

Connection继承自std::enable_shared_from_this，也就意味着外部我们会操作其shared_ptr<Connection>，libtnet几乎所有的对象都采用智能指针的方式来进行内存管理。

当Connection创建成功之后，会通过IOLoop的addHandler接口，将其绑定到ioloop上面：

    ConnectionPtr_t conn = shared_from_this();
    m_loop->addHandler(m_fd, TNET_READ, std::bind(&Connection::onHandler, conn, _1, _2));

因为我们直接在std::bind里面使用了shared_ptr，所以ioloop自然引用了该Connection，外部不需要在存储Connection，以防内存泄露。

对于一个connection而言，它只可能有几种状态，

- Connecting，表明正在尝试连接，发生在connect返回EINPROGRESS。
- Connected，连接已经建立成功，发生在connect成功或者accept成功。
- Disconnecting，表明连接正在断开，发生在用户主动调用shutDown之后。
- Disconnected，连接已经断开，这时候对应的socket也会被close掉。

# Event Callback

在Connection中，我们使用一个event callback来绑定相应事件的回调。主要有如下connection event：

- Conn_EstablishedEvent, 当server accept成功，创建了Connection对象之后，触发。
- Conn_ConnectEvent，当client connect成功，触发。
- Conn_ConnectingEvent，当client connect返回EINPROGRESS，触发。
- Conn_ReadEvent，当连接可读，触发。
- Conn_WriteCompleteEvent，当发送的数据都发送完毕，触发。
- Conn_ErrorEvent，当连接有错误发生，触发。
- Conn_CloseEvent，当连接主动或者被动关闭，触发。

event callback原型如下：

    typedef shared_ptr<Connection> ConnectionPtr_t;
    typedef std::function<void (const ConnectionPtr_t&, ConnEvent, const void* context)> ConnEventCallback_t;
    
对应不同的事件，触发的时候context的内容不同。现阶段，只有ReadEvent的时候context为StackBuffer，原型如下：

    class StackBuffer
    {
    public:
        StackBuffer(const char* buf, size_t c) : buffer(buf), count(c) {}
        
        const char* buffer;
        size_t count;    
    };
    
当连接可读的时候，Connection会将数据读取到栈上面，并用StackBuffer来指代，这样当外部处理ReadEvent的时候就能通过将context转换成StackBuffer获取到读取的数据。

下面简单说明一下一些设计上面的取舍：

- 为什么只提供一个event callback，而不提供read callback，write complete callback，close callback多个回调接口？
    
    libtnet的所有callback都采用的是std::function实现，而该对象占用32字节，如果每个event都提供一个对应的callback，那么内存的开销会有点大，同时大部分时候很多callback我们是不感兴趣的。
    
    还有一个重要的原因在于只提供一个event callback，外部的一些对象就可以通过该callback跟Connection绑定，也就是将其自身的生命周期与Connection绑定在了一起，当Connection删除的时候该对象也自行删除。libtnet中，HttpConnection，WsConnection都是采用该方法，因为对于一个Http连接来说，如果底层的Tcp连接都断开无效了，基于Tcp的Http连接自然就无效了。
    
- Connection为什么不缓存读取的数据，而是交由外部callback去处理？

    Connection作为一个底层的类，对于读取的数据，并不知道具体需要如何处理，所以还不如将数据直接发到外层，供上层实际的应用逻辑处理。但是如果后续Connection考虑支持ssl，那么就需要进行缓存数据了。
    
# Write

Connection建立之后，默认只会在ioloop中设置TNET_READ事件，因为epoll采用的水平触发模式，如果直接设置TNET_WRITE事件，那么epoll会一直通知socket可写，但实际上并没有可以发送的数据。

所以，libtnet采用如下的方式进行数据发送：

- 直接调用writev函数进行数据发送
- 如果数据未发送完毕，则向ioloop注册TNET_WRITE事件，下次触发可写的时候继续发送，直至发送成功，清除TNET_WRITE事件。

另外，在发送的时候，我们还需要考虑signal pipe的情况，所以需要忽略该singal。使用如下方式：

    class IgnoreSigPipe
    {
    public:
        IgnoreSigPipe()
        {
            signal(SIGPIPE, SIG_IGN);    
        }    
    };

    static IgnoreSigPipe initObj;

当libtnet启动的时候，就忽略了signal pipe信号。虽然这样做稍微有一点副作用，但大部分时候我们并不需要关注SIGPIPE信号。

# Kick Off Connection

通常，为了处理不活跃连接，程序都会将每个connection设置一个timer，如果timer到了该连接仍然没有交互，则会删除该连接，否则则继续更新timer。另一种做法就是提供一个time wheel，将connection放置在该wheel中，如果有交互，则在wheel中移动。

libtnet采用了一种更简单，但是精度比较差的做法。

当server成功创建一个connection之后，将会添加到一个ConnChecker中，checker保存的是该connection的weak_ptr。每隔一段时间，checker检查一批connection：

- 如果connection weak_ptr无法lock提升至shared_ptr，证明该连接已经删除，checker直接移除。
- 如果connection处于connecting状态，并且超过了设置的最大连接超时时间，shutDown该connection。
- 如果connection处于connected状态，并且在一段时间内没有任何交互，shutDown。

ConnChecker的检查间隔以及每次检查步数都可以通过外部设置。使用ConnChecker虽然简单，但是在连接数过大的情况下面，一些过期的connection不能立刻被清理掉。对于这个问题，我觉得可以接受，一个连接一秒之后被关闭还是两秒之后被关闭，差别真的不大。如果我们真的需要对一些connection做精确的时间控制，那直接可以对其使用timer。
