最近在用go语言做一个挂载大量长连接的推送服务器，虽然已经完成，但是内存占用情况让我不怎么满意，于是考虑使用libtnet来重新实现一个。后续我会使用comet来表明推送服务器。

对于comet来说，单机能支撑大量的并发连接，是最优先考虑的事项。虽然现在业界已经有了很多数据，说单机支撑200w，300w，但我还是先把目标定在100w上面，主要的原因在于实际运行中，comet还会有少量逻辑功能，我得保证在单机挂载100w的基础上，完全能无压力的处理这些逻辑。

# CometServer Test

首先我用libtnet简单写了一个comet server。它接受http请求，并将其挂起，过一段随机时间之后在返回200。

    void onTimeout(const TimingWheelPtr_t& wheel, const WeakHttpConnectionPtr_t& conn)
    {
        HttpConnectionPtr_t c = conn.lock();
        if(c)
        {
            c->send(200);
        } 
    }
    
    void onHandler(const HttpConnectionPtr_t& conn, const HttpRequest& request)
    {
        int timeout = random() % 60 + 30;
        comet.wheel->add(std::bind(&onTimeout, _1, WeakHttpConnectionPtr_t(conn)), timeout * 1000);
    }
    
    int main()
    {
        TcpServer s;        
        s.setRunCallback(std::bind(&onServerRun, _1));
        HttpServer httpd(&s);
        httpd.setHttpCallback("/", std::bind(&onHandler, _1, _2));
        httpd.listen(Address(11181));
        s.start(8);
        return 0; 
    }


可以看到comet server只是负责了挂载长连接的事情，而没有消息的推送。在实际项目中，我已经将挂载连接和推送消息分开到两个服务去完成。所以这里comet仅仅是挂载连接测试。

# 测试机器准备

因为linux系统上面一个网卡tcp连接端口数量是有限制的，我们调整ip_local_port_range使其能支撑60000个tcp连接：

    net.ipv4.ip_local_port_range = 1024 65535

对于100w连接来说，我们至少需要16台机器，但实际我只有可怜的3台4G内存的虚拟机。所以就要运维的童鞋在每台机器上面装了6块网卡。这样我就能建立100w的连接了。

测试客户端也非常简单，每秒向服务器请求1000个连接，但是需要注意的是，因为一台机器上面有多块网卡，所以在创建socket之后，我们需要将socket绑定到某一块网卡上面。

实际测试中，因为内存问题，每台机器顶多能支撑34w左右的tcp连接，对我来说已经足够，所以也懒得去调优了。

# CometServer Linux调优

首先我们需要调整最大打开文件数，在我的机器上面，nr_open最大的值为1048576，对我来说已经足够，所以我将最大文件描述符数量调整为1040000。

    fs.file-max = 1040000

然后就是对tcp一些系统参数的调优：

    net.core.somaxconn = 60000
    net.core.rmem_default = 4096
    net.core.wmem_default = 4096
    net.core.rmem_max = 16777216
    net.core.wmem_max = 16777216
    net.ipv4.tcp_rmem = 4096 4096 16777216
    net.ipv4.tcp_wmem = 4096 4096 16777216
    net.ipv4.tcp_mem = 786432 2097152 3145728
    net.core.netdev_max_backlog = 60000
    net.ipv4.tcp_fin_timeout = 15
    net.ipv4.tcp_max_syn_backlog = 60000
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.tcp_max_orphans = 131072
    net.ipv4.tcp_keepalive_time = 1200
    net.ipv4.tcp_max_tw_buckets = 60000
    net.netfilter.nf_conntrack_max = 1000000
    net.netfilter.nf_conntrack_tcp_timeout_established = 1200

对于如何调节这些值，网上都是各有各的说法，建议直接man 7 tcp。我在实践中也会通过查看dmesg输出的tcp错误来动态调节。这里单独需要说明的是tcp buffer的设置，我最小和默认都是4k，这主要是考虑到推送服务器不需要太多太频繁的数据交互，所以需要尽可能的减少tcp的内存消耗。

# 测试结果

实际的测试比较让我满意。

cometserver test8个进程cpu消耗都比较低，因为有轮训timing wheel然后再发送200的逻辑，所以铁定有cpu消耗，如果只是挂载，cpu应该会更低。

    PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND                                                                                                       
     4685 root      20   0  187m 171m  652 S 19.9  1.1   2:19.20 cometserver_test                                                                                                
     4691 root      20   0  191m 175m  656 S 16.6  1.1   2:17.80 cometserver_test                                                                                                
     4686 root      20   0  170m 155m  652 S 16.3  1.0   2:09.54 cometserver_test                                                                                                
     4690 root      20   0  183m 167m  652 S 16.3  1.1   2:11.44 cometserver_test                                                                                                
     4692 root      20   0  167m 152m  652 S 16.3  1.0   2:11.29 cometserver_test                                                                                                
     4689 root      20   0  167m 152m  652 S 15.3  1.0   2:03.08 cometserver_test                                                                                                
     4687 root      20   0  173m 158m  652 S 14.3  1.0   2:07.34 cometserver_test                                                                                                
     4688 root      20   0  129m 114m  652 S 12.3  0.7   1:35.77 cometserver_test
    
socket的统计情况：
    
    [root@localhost ~]# cat /proc/net/sockstat
    sockets: used 1017305
    TCP: inuse 1017147 orphan 0 tw 0 alloc 1017167 mem 404824
    UDP: inuse 0 mem 0
    UDPLITE: inuse 0
    RAW: inuse 0
    FRAG: inuse 0 memory 0
    
可以看到，总共有1017147个tcp链接，同时占用了将近4G（1017167是页数，需要乘以4096）的内存。
    
系统内存的情况：

    [root@localhost ~]# free
                 total       used       free     shared    buffers     cached
    Mem:      16334412   11210224    5124188          0     179424    1609300
    -/+ buffers/cache:    9421500    6912912
    Swap:      4194296          0    4194296
    
系统有16G内存，还有5G可用，所以不出意外单机应该还能承载更多的tcp连接。

# 总结

使用libtnet开发的一个简单的comet server支撑了百万级的连接，加深了我对其应用的信心。

libnet地址[https://github.com/siddontang/libtnet](https://github.com/siddontang/libtnet)，欢迎围观。