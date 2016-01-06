libtnet采用的是prefork + event loop的架构方式，prefork就是server在启动的时候预先fork多个子进程同时工作，而event loop则是基于epoll的事件处理机制。

在最新的linux系统中，提供了timerfd，eventfd，signalfd，加上原先的socket，大部分功能都可以抽象成io事件来处理了。而在libtnet中，这一切的基础就是IOLoop。

类似于tornado，libtnet的IOLoop也提供了相似的接口。其中最核心的就是以下三个：

    typedef std::function<void (IOLoop*, int)> IOHandler_t;

    int addHandler(int fd, int events, const IOHandler_t& handler);
    int updateHandler(int fd, int events);
    int removeHandler(int fd);  

对于任意的IO，我们可以注册感兴趣的事件（TNET_READ和TNET_WRITE），并绑定一个对应的callback回调。

callback的回调采用的是std::function的方式，这也就意味着，你可以在外部通过std::bind绑定任意不同的实现，再加上shared_ptr技术模拟闭包。

假设现在我们需要创建了一个socket对象，并将其添加到IOLoop中，我们可以这么做：

    std::shared_ptr<Connection> conn = std::make_shared<Connection>(socketfd);
    
    ioloop->addHandler(socketfd, TNET_READ, std::bind(&Connection::onHandler, conn, _1, _2));
    
这样，当该socket有读事件发生的时候，对应的onHandler就会被调用。在这里，我是用了shared_ptr技术，主要是为了方便进行对象生命周期的管理。

在上面的例子中，因为std::bind的时候引用了conn，只要不将socketfd进行removeHandler，conn对象就会一直存在。所以libtnet在IOLoop内部，自行维护了conn对象的生命周期。外面不需要在将其保存到另一个地方（如果真保存了该shared_ptr的conn，反而会引起内存泄露）。在libtnet的基础模块中，我都使用的是weak_ptr来保存相关对象，每次使用都通过lock来判定是否该对象存活。

在IOLoop内部，我使用一个vector来存放注册的handler，vector的索引就是io的fd。这样，我们通过io的fd就可以非常快速的查找到对应的handler了。为什么可以这样设计，是因为在linux系统中，进程中新建文件的file descriptor都是系统当前最小的可用整数。譬如，我创建了一个socket，fd为10，然后我关闭了该socket，再次新建一个socket，这时候新的socket的fd仍然为最小可用的整数，也就是10。

# EPoll

提到linux下面的高性能网络编程，epoll是一个铁定绕不开的话题，关于epoll的使用，网上有太多的讲解，这里就不展开了。

libtnet在Poller中集成了epoll，参考了libev的实现。epoll有两种工作模式，水平触发和边沿触发，各有利弊。libtnet使用的是水平触发方式，主要原因在于水平触发方式在有消息但是没处理的时候会一直通知你处理，实现起来不容易出错，也比较简单。

## fork and epoll_create

这里顺便记录一下我在实现prefork模型的时候遇到的一个坑。这个问题就是epoll fd应该在fork之前还是之后创建？

大家都知道，linux fork的时候采用COW（copy on write）方式复制父进程的内容，然后我想当然的以为各个子进程会拥有独立的epoll内核空间，于是在fork之前创建了epoll fd。但是后面我却惊奇的发现一个子进程对epoll的操作竟然会影响另一个子进程。也就是说，各个子进程共享了父进程的epoll内核空间。

所以，epoll fd的创建应该在fork之后，各个子进程独立创建。

# Example

## Timer

IOLoop提供了一个简单的runAfter函数，用以实现定时器功能，使用非常简单：

    void func(IOLoop* loop)
    {
        cout << "hello world" << endl;
        loop->stop();
    }

    IOLoop loop;
    loop.runAfter(10 * 1000， std::bind(&func, &loop));
    loop.start();
    
loop启动十秒之后，会打印hello world，然后整个loop退出。更多定制化的timer使用，可以使用libtnet提供的Timer class。

## Callback

libtnet是一个单线程单ioloop的模型，但是不排除仍然会有其他线程希望与IOLoop进行通信，所以IOLoop提供了addCallback功能，这是libtnet唯一一个线程安全的函数。因为加入callback是一个很快速的操作，IOLoop使用了spinlock。在IOLoop每次循环的末尾，会将全部的callback取出，依次执行。

    void callback(IOLoop* loop)
    {
        cout << "tell to exit" << endl;
        loop->stop();
    }

    IOLoop loop;
    loop.addCallback(std::bind(&func, &loop));
    loop.start();
    

