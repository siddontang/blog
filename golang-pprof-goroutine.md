有一段时间，我们的推送服务socket占用很不正常，我们自己统计的同时在线就10w的用户，但是占用的socket竟然达到30w，然后查看goroutine的数量，发现已经60w+。

每个用户占用一个socket，而一个socket，有read和write两个goroutine，简化的代码如下：

    c, _ := listerner.Accept()
    
    go c.run()
    
    func (c *conn) run() {
        go c.onWrite()
        c.onRead()
    }
    
    func (c *conn) onRead() {
        stat.AddConnCount(1)
        
        //on something
        
        stat.AddConnCount(-1)
        
        //clear
        //notify onWrite to quit
    }
    
当时我就怀疑，用户同时在线的统计是正确的，也就是之后的clear阶段出现了问题，导致两个goroutine都无法正常结束。在检查代码之后，我们发现了一个可疑的地方，因为我们不光有自己的统计，还会将一些统计信息发送到我们公司的统计平台，代码如下：

    ch = make([]byte, 100000)
    func send(msg []byte) {
        ch <- msg
    }
    
    //在另一个goroutine的地方，
    msg <- msg
    httpsend(msg)
    
我们channel的缓存分配了10w，如果公司统计平台出现了问题，可能会导致channel阻塞。但到底是不是这个原因呢？

幸运的是，我们先前已经在代码里面内置了pprof的功能，通过pprof goroutine的信息，发现大量的goroutine的当前运行函数在httpsend里面，也就是说，公司的统计平台在大并发下面服务不可用，虽然我们有http超时的处理，但是因为发送的数据量太频繁，导致整体阻塞。

临时的解决办法就是关闭了统计信息的发送，后续我们会考虑将其发送到自己的mq上面，虽然也可能会出现mq服务不可用的问题，但是说句实话，比起自己实现的mq，公司的统计平台更让我不可信。

这同时也给了我一个教训，访问外部服务一定要好好处理外部服务不可用的问题，即使可用，也要考虑压力问题。

对于pprof如何查看了goroutine的问题，可以通过一个简单的例子说明:

    package main

    import (
        "net/http"
        "runtime/pprof"
    )
    
    var quit chan struct{} = make(chan struct{})
    
    func f() {
        <-quit
    }
    
    func handler(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "text/plain")
        
        p := pprof.Lookup("goroutine")
        p.WriteTo(w, 1)
    }
    
    func main() {
        for i := 0; i < 10000; i++ {
            go f()
        }
        
        http.HandleFunc("/", handler)
        http.ListenAndServe(":11181", nil)
    }

这上面的例子中，我们启动了10000个goroutine，并阻塞，然后通过访问http://localhost：11181/，我们就可以得到整个goroutine的信息，仅列出关键信息：

    goroutine profile: total 10004
    
    10000 @ 0x186f6 0x616b 0x6298 0x2033 0x188c0
    #	0x2033	main.f+0x33	/Users/siddontang/test/pprof.go:11
    
可以看到，在main.f这个函数中，有10000个goroutine正在执行，符合我们的预期。

在go里面，还有很多运行时查看机制，可以很方便的帮我们定位程序问题，不得不赞一下。