# 前言

对于高性能web服务器来说，一个好的网络模型是必不可少的，这样能让程序员专注于上层业务逻辑的开发，而不需要过多的关注底层网络的交互。虽然网上现在有很多介绍高性能网络架构的文章，但是这里我还是想自己结合多年的开发经验总结一下，也当算是自己知识的沉淀。

测试环境

- os: mac os 10.7.4
- cpu: intel core i5 1.7 GHz
- mem: 4G 1600 MHz DDR3

开发工具

- python: 2.7

测试工具

- tool: http-load

这里，需要说明几个问题

- 只考虑单机，不提整体分布式。
- os的选择，unix/linux都可以，而windows则没有考虑，虽然它有很强的iocp机制用来高效的处理网络。
- 语言的选择，对于io密集型的程序，语言的差别是很小的，c/c++在碰到io操作的时候性能不见得比python高多少，而python的开发非常的快速，所以我选择了python。但如果是cpu密集型程序，c/c++那真还是王道了。
- 测试工具的选择，虽然有很多测试web服务器性能的工具，之所以选择http-load，主要还是在于它足够简单，而且我只需要进行大概比较，来说明网络模型的性能，所以当然是越简单越好。


# 一个简单的web server

首先我们来看一个简单的web server例子

    import socket

    HOST = '127.0.0.1'                 
    PORT = 8888              

    msg = '200 OK HTTP/1.1\r\n\r\nHello World'

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind((HOST, PORT))
    s.listen(1)

    def hello(conn):
        conn.recv(1024)
        
        conn.sendall(msg)
        conn.shutdown(socket.SHUT_RDWR)
        conn.close()

    while 1:
        conn, addr = s.accept()
        hello(conn)

上面是一个简单的web server，它接受client的连接，读取client的数据，并且发回一个hello world，就这么简单。不过它的缺点也很明显，只能处理单个连接，不能同时处理多个连接。也就是说，这个server处理不了并发。

# blocking io + multi process

为了解决上面的问题，我们引入process或者thread，也就是说，当server accept一个client请求之后，会将其扔到一个新的进程或者线程中去，而主循环继续处理下一个连接请求，这样就能保证并发。

在早期的unix系统中，推荐使用fork模型，也就是将socket扔入一个新创建的进程中。因为在unix下面，fork很迅速，并且开销很小。

    while 1:
        conn, addr = s.accept()

        if os.fork() > 0:
            conn.close()
        
            continue

        hello(conn)

        os._exit(0)

当accept成功之后，server使用fork创建一个子进程，然后父进程继续监听网络连接。而后续所有与client的交互都是在子进程里面进行，这样就有了并发。

这里需要注意一点，mac下面系统默认max user processes是一个很小的数，我的为709，如果不能调高，很容易就会因为fork太多导致Resource temporarily unavailable，但是mac下面坑爹的是ulimit -u竟然无效，google了一下找到一些[解决方案](http://blog.ghostinthemachines.com/2010/01/19/mac-os-x-fork-resource-temporarily-unavailable/)。

补充一点，上面这个程序即使把process的数量设置的在高，也会造成process资源的耗尽，原因在于僵尸进程，具体可以参考[两次fork](http://blog.csdn.net/yygydjkthh/article/details/7342160)，以及[python-and-parallel-programming](http://mildopinions.wordpress.com/2008/05/31/python-and-parallel-programming/)

对于fork，为了防止僵尸进程问题，开始我使用注册singal的方式处理，如下

    def handleSIGCHLD(num, stackframe):
        if num == signal.SIGCHLD:
        try:
            (pid, status) = os.waitpid(-1, os.WNOHANG)
        except:
            pass


    signal.signal(signal.SIGCHLD, handleSIGCHLD)


但是却发现在socket accept的时候，老是出现
    
    socket.error: [Errno 4] Interrupted system call

这样的错误，原因在于socket的系统调用可能会被signal打断，不过对于信号这个东西，本来我也不怎么感冒，就不想深究了。所以有了后一种简单的做法，就是在accept之前waitpid一下，这样就能回收一些僵尸进程，只不过可能还是有一些进程无法回收，不过觉得也能接受了。对于二次fork解决方案，感觉fork的开销还是有的，这里就不怎么想用了。

# blocking io + multi thread

对于thread，代码差别不大，只需要将首发数据的功能移到thread里面

    def callback(conn):
        hello(conn)

    while 1:
        conn, addr = s.accept()

        th = threading.Thread(target=callback, kwargs = {'conn':conn})
        th.start()


# multi process vs multi thread

网上有太多分析多进程和多线程的文章，譬如这篇[Forking vs Threading](http://www.geekride.com/fork-forking-vs-threading-thread-linux-kernel/)，而我本机的测试结果如下：


    siddontang:~/repository/perfsvr $ http_load -p 10 -fetches 5000 url.list 
    5000 fetches, 10 max parallel, 55000 bytes, in 1.55861 seconds
    11 mean bytes/connection
    3207.98 fetches/sec, 35287.8 bytes/sec
    msecs/connect: 0.265093 mean, 20.846 max, 0.197 min
    msecs/first-response: 2.78085 mean, 23.332 max, 0.584 min
    HTTP response codes:

    siddontang:~/repository/perfsvr $ http_load -p 10 -fetches 5000 url.list 
    5000 fetches, 10 max parallel, 55000 bytes, in 2.73079 seconds
    11 mean bytes/connection
    1830.97 fetches/sec, 20140.7 bytes/sec
    msecs/connect: 0.244044 mean, 10.649 max, 0.063 min
    msecs/first-response: 5.12083 mean, 12.943 max, 0.436 min
    HTTP response codes:


第一个测试结果是thread的，第二个为process，可以看到，thread的性能要好于process，不过也不能因此下结论multi thread模型就更好，因为我这个只是简单的测试。

之所以要写multi process/thread主要是因为，在上面这两种模型下面，如果并发量大的时候，都会遇到一个比较严重的问题，就是系统切换开销以及内存压力。这里有一个[benchmark](http://bulk.fefe.de/scalability/)。


# non blocking socket

上面说的网络模型都是基于blocking socket，所谓blocking socket就是当socket在send或者recv的时候，进程/线程会一直block直到io完成。我们之所以会对于每一个连接启用一个进程/线程，就是不能因为io的block导致整个系统的block。而non blocking socket则没有这种问题，当一个socket是non blocking的时候，它在调用send，recv等时候可能会立即返回而不作任何事情，而后续的事情我们通过其他机制获得并处理。这样就不会因为io的block造成整个程序的block。

# select/epoll/kqueue

对于non blocking socket，我们可以使用select等进行监听，当这个socket可读，可写等状态的时候，select会触发相应的回调进行处理。同时，select可以监听多个non blocking socket，这样就达到了多路复用io的效果。而对于epoll，kqueue，大概原理一样，只是在实现上面不一样罢了。

这里，我决定开始直接使用tornado的源码，这里不得不佩服tornado的强悍，一个ioloop就封装好了所有的东西，并且可以随时切换。因为在mac，所以只能使用select和kqueue。

## select

在tornado里面使用select很简单，需要注意两点：

- 通过IOLoop传入特定的poll机制。
- application的listen里面会使用IOLoop的单件，所以生成的ioloop需要install一下  

代码如下：

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("Hello, world")

    application = tornado.web.Application([
        (r"/", MainHandler),
    ])

    loop = tornado.ioloop.IOLoop(tornado.ioloop._Select())
    loop.install()
    application.listen(8888)
    loop.start()

使用http_load的测试结果，可以看到，在同时100并发的情况下面，性能很不错。而先前multi process/thread模型在我的机器上面当并发量为100的时候已经有很多请求处理不过来直接报错了。

    siddontang:~/repository/perfsvr $ http_load -p 100 -fetches 5000 url.list 
    5000 fetches, 100 max parallel, 60000 bytes, in 2.05569 seconds
    12 mean bytes/connection
    2432.28 fetches/sec, 29187.3 bytes/sec
    msecs/connect: 0.192841 mean, 3.623 max, 0.082 min
    msecs/first-response: 40.5787 mean, 105.812 max, 15.746 min
    HTTP response codes:
    code 200 -- 5000

再来说说select的使用方式。select会关心3个队列，read，write和error，对于一个socket而言，它如果对某一种事件感兴趣，就加入到相应的队列里面去。譬如一个socket这时候只关心是否能read，那它就加入read队列。而select则在每次运行的时候，依次去遍历这些队列，看里面哪些socket准备就绪可以操作了，如果有将他加入一个可处理的列表里面去。tornado下面select使用代码如下：

    def poll(self, timeout):
        readable, writeable, errors = select.select(
            self.read_fds, self.write_fds, self.error_fds, timeout)
        events = {}
        for fd in readable:
            events[fd] = events.get(fd, 0) | IOLoop.READ
        for fd in writeable:
            events[fd] = events.get(fd, 0) | IOLoop.WRITE
        for fd in errors:
            events[fd] = events.get(fd, 0) | IOLoop.ERROR
        return events.items()


## kqueue

让tornado支持kqueue模式很简单，mac下面就自动就是的，不过为了与select对比，我们使用如下方式

    loop = tornado.ioloop.IOLoop(tornado.ioloop._Kqueue())
    
测试结果如下

    siddontang:~/repository/perfsvr $ http_load -p 100 -fetches 5000 url.list 
    5000 fetches, 100 max parallel, 60000 bytes, in 2.44096 seconds
    12 mean bytes/connection
    2048.38 fetches/sec, 24580.5 bytes/sec
    msecs/connect: 7.95017 mean, 32.623 max, 0.49 min
    msecs/first-response: 40.0568 mean, 73.247 max, 14.235 min
    HTTP response codes:
    code 200 -- 5000

可以看到，kqueue的结果也是挺不错的，能够支持高并发量的处理，但是与select对比发现，select的性能竟然比kqueue要高，这个在某些时候是的，我后续在说明。

这里先说明一下kqueue的工作方式，相比于select，我感觉kqueue强大了很多，对于kqueue的设计，具体可以参考这篇[paper](https://docs.google.com/viewer?url=http%3A%2F%2Fpeople.freebsd.org%2F~jlemon%2Fpapers%2Fkqueue.pdf)。kqueue不光可以处理socket的事件，同时还能处理异步io，信号，文件变化等，感觉就是事件监听一锅端，不过接口还是挺好理解的。

kqueue有两个部分，kqueue和kevent。kqueue主要是用来描述event的队列，它通过control这个函数来获取当前有变化的event。而kevent则是对应的相应事件。

kevent的[详细说明](http://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2)，如果只是监听socket的，我们需要关注如下几个东西:

- ident kevent的一个唯一标识符，一般我们就用需要监控的文件句柄表示
- filter 过滤器，一般关注KQ_FILTER_READ和KQ_FILTER_WRITE，即可读可写。
- flag filter之后需要关注的action，一般为KQ_EV_ADD（添加），KQ_EV_DELETE（删除），KQ_EV_EOF（EOF）和KQ_EV_ERROR（出错）

kqueue的poll代码如下：
    
    def poll(self, timeout):
        kevents = self._kqueue.control(None, 1000, timeout)
        events = {}
        for kevent in kevents:
            fd = kevent.ident
            if kevent.filter == select.KQ_FILTER_READ:
                events[fd] = events.get(fd, 0) | IOLoop.READ
            if kevent.filter == select.KQ_FILTER_WRITE:
                if kevent.flags & select.KQ_EV_EOF:
                    events[fd] = IOLoop.ERROR
                else:
                    events[fd] = events.get(fd, 0) | IOLoop.WRITE
            if kevent.flags & select.KQ_EV_ERROR:
                events[fd] = events.get(fd, 0) | IOLoop.ERROR
        return events.items()


## epoll

对于epoll，因为我本机没有环境无法测试，加之网上介绍epoll的资料太多了，就不多做介绍了。这里需要关注它的两种模式，edge trigger和level trigger，通俗点说，就是如下：

- edge trigger：有了消息，通知你一次，你不处理完成，拉倒，再也不通知你了，直到下次来消息。
- level trigger：有了消息，通知你，你不处理，一直通知你，直到你处理完。

kqueue也有这两种模式，通过KQ_EV_CLEAR来设置，这里就不深入讨论了。

## why kqueue and epoll are better

说完了这三种实现，现在来探讨一个问题，为什么kqueue和epoll比select好。从机制上面说，kqueue和epoll只关注有事件发生的socket，而select则需要将所有的注册socket遍历一遍，光从每次遍历的socket对象来说，select就要比kqueue和epoll多查看几个。

所以，在连接数较多并且很多socket不是特别活跃的时候，select的性能是有问题的，但是如果在连接数较少并且socket活跃的时候，select的性能很不错，并且不必kqueue和epoll差。

这是从对socket遍历机制上面说，为什么kqueue和epoll性能高。但如果在深入一点呢？

### epoll/kqueue实现浅析

我们来想想，如果要实现一个类似epoll的功能怎么做呢？

- 需要一个数据结果来保存整个注册的socket，而且要非常快速的进行socket的插入，删除操作，这个使用红黑树是一个很不错的想法，性能为O(lgn)。
- 当一个socket有事件发生的时候，系统中断会通过描述符找到对应的socket，然后我们将其加入一个队列里面去
- 外部拿到这个可处理队列就可以进行操作，如果设置为level。 trigger，那么处理一轮过后，检查对应的socket是否还有东西需要处理，有的话则继续加入队列，如果为edge trigger则不管了。

照理说，kqueue应该也实现同样的功能，但从这篇[paper](https://docs.google.com/viewer?url=http%3A%2F%2Fpeople.freebsd.org%2F~jlemon%2Fpapers%2Fkqueue.pdf)可以看出，对于socket的保存，kqueue直接使用了一个array，就跟系统的open file table一样，为什么这么做呢？因为对于进程来说，它都有一个max file的最大值，譬如1024，所以分配文件的描述符最大是不会超过1024的，而且类unix系统上面分配的文件描述符都默认取得是最小的一个可用的，基于这些，我们就可以使用一个array来进行索引。对于这点[picoev](http://developer.cybozu.co.jp/kazuho/2009/08/picoev-a-tiny-e.html)这个网络框架库可算是用到了极致。后续有机会在介绍。


# 多路复用socket + thread pool for logic

上面介绍了多路复用IO，那么当然我们就要考虑使用这个技术来搭建服务器了。以kqueue为例，我们得面对一个问题，虽然系统处理网络的能力提升了，但是如果收到数据之后我们需要进行一个费时的逻辑操作，仍然会导致整体的性能下降。所以我们就需要一套机制来分解网络和逻辑。

一个比较好的做法就是网络使用一个线程，而逻辑处理使用另一个线程，两个线程之间通过消息队列进行交互。这样当网络线程收到消息处理之后，它只需要将socket的fd以及对应的信息放进消息队列里面，就可以继续处理下一个网络请求了。而逻辑线程则从队列里面取出消息，处理，完成之后将fd以及对应的返回消息放到一个队列里面，让网络线程取出处理就可以了。因为逻辑处理算是一个比较消耗的操作，一般会启用多个逻辑线程，也就是用thread pool进行管理。

这个模型是不错的，那么如果网络线程压力顶不住了，怎么办呢？一般的做法也是开启多个线程用于处理网络，每个线程listen同一个端口，这样就可以进行负载均衡，而每个网络线程仍然对应一批逻辑线程。

网络线程与逻辑线程之间的交互通过消息队列是一个比较好的方式，但因为涉及到多线程，多消息队列的操作仍然会涉及到锁的操作，所以有时候还会其它的一些方式。

## 多队列缓冲机制

这个机制是我以前做游戏服务器的时候跟同事弄出来的一个东西，不知道业界有没有同样的做法的。流程如下:

- 网络线程自己维护一个队列，新收到了消息就加入队列，这个是没有任何锁开销的
- 到一定阶段，譬如网络线程循环转了几圈，或者消息队列达到一定数量，则将该消息队列splice到一个中间队列，这个是需要锁保护的，而且这个只会涉及到指针的交互，所以会很快速
- 逻辑线程将中间队列splice到自己的消息队列种，同时将中间队列置空，这样逻辑线程就能拿到整个消息队列了，这个也是需要锁保护的，同理，只会涉及到指针的操作，会很快速
- 逻辑线程依次取出消息处理，这个也就没有任何锁开销了

可以看出，这套机制是以降低消息的及时性为代价的，不过鉴于消息本来就是异步的，线程级别之间稍微的延后跟网络比起来我觉得问题还不大。

## 无锁的ring buffer

无锁的ring buffer这种结构是我在另一家公司接触到的技术，原理如下：

- 估算分配一块内存用作ring buffer，这块区域是不会在变化的了
- 线程A只会往ring buffer里面写数据，使用tail来表明下一个可写入位置，线程B只会从里面读数据，使用head来表明下一个可读取的位置
- 当head = tail的时候，认为buffer为空，没有数据
- 当tail + 1 = head的时候，认为队列满了，无法在写入数据，为什么要留出一个元素空间呢，因为如果不留出，那么buffer满也是tail = head，而这个就无法跟buffer为空区分了。
- 线程A写入数据的时候，首先判断buffer是否有足够的空间写入，如果能写入，就写入数据，同时更新tail的位置
- 线程B读取数据的时候，首先判断buffer是否有足够的数据可读，如果能读取，则读取数据，同时更新head的位置

这里，可能会有人疑惑，为什么这个是线程安全的？

- 首先，我们知道，对于head和tail数值的读取是线程安全的，为什么这么说，在c种，head和tail就是指针，而对于指针的读取，是一个原子操作，这个是线程安全的。
- 以写入为例，线程A首先会读取head和tail的当前值，后续即使线程B修改了head的值，也不会影响这次的写入操作。因为B修改了head，表明A可以写入更多数据，但A获取到的head仍然是先前的值，所以能写入的数据就会按照先前的head计算。当A写入数据之后，更新tail的值。读取跟写入的原理一样。

而这个在linux内核里面是有实现的，可以看[这里](http://www.ibm.com/developerworks/cn/linux/l-cn-lockfree/index.html).

# 总结

写了这么多字，感觉可以手工了。这只是我个人经验，可能还会有更好的做法。对于高性能服务器而言，网络只是需要考虑的一个方面，还会有很多问题需要考虑。其实更需要关注的是整体架构以及整体的性能调优。

另外，感觉还缺少了很多图片，总觉得纯靠文字很多问题还是说的不够深入，后续还是把图片给补上。另外在搭配一个ppt，当然准备使用express.js做，感觉就完美了。

不过，总觉得还是有很多东西可以说的，对于网络这块，我还会继续深入研究，譬如底层的tcp/ip协议，p2p等。希望到时候能有更好的总结。