虽然写出7x24小时不间断运行的服务是一件很酷的事情，但是我们仍然在某些时候，譬如服务升级，配置更新等，得考虑如何优雅的结束这个服务。

当然，最暴力的做法直接就是`kill -9`，但这样直接导致的后果就是可能干掉了很多运行到一半的任务，最终导致数据不一致，这个苦果只有遇到过的人才能深深地体会，数据的修复真的挺蛋疼，有时候还得给用户赔钱啦。

所以，通常我们都是给服务发送一个信号，SIGTERM也行，SIGINTERRUPT也成，反正要让服务知道该结束了。而服务收到结束信号之后，首先会拒绝掉所有外部新的请求，然后等待当前所有正在执行的请求完成之后，在结束。当然很有可能当前在执行一个很耗时间的任务，导致服务长时间不能结束，这时候就得决定是否强制结束了。

具体到go的HTTP Server里面，如何优雅的结束一个HTTP Server呢？

首先，我们需要显示的创建一个listener，让其循环不断的accept新的连接供server处理，为啥不用默认的http.ListenAndServe，主要就在于我们可以在结束的时候通过关闭这个listener来主动的拒绝掉外部新的连接请求。代码如下:

```
l, _ := net.Listen("tcp", address)
svr := http.Server{Handler: handler}
svr.Serve(l)
```

Serve这个函数是个死循环，我们可以在外部通过close对应的listener来结束。

当listener accept到新的请求之后，会开启一个新的goroutine来执行，那么在server结束的时候，我们怎么知道这个goroutine是否完成了呢？

在很早之前，大概go1.2的时候，笔者通过在handler入口处使用sync WaitGroup来实现，因为我们有统一的一个入口handler，所以很容易就可以通过如下方式知道请求是否完成，譬如：

```
func (h *Handler) ServeHTTP(w ResponseWriter, r *Request) {
    h.svr.wg.Add(1)
    defer h.svr.wg.Done()

    ......
}
```

但这样其实只是用来判断请求是否结束了，我们知道在HTTP 1.1中，connection是能够keepalived的，也就是请求处理完成了，但是connection仍是可用的，我们没有一个好的办法close掉这个connection。不过话说回来，我们只要保证当前请求能正常结束，connection能不能正常close真心无所谓，毕竟服务都结束了，connection自动就close了。但谁叫笔者是典型的处女座呢。

在go1.3之后，提供了一个ConnState的hook，我们能通过这个来获取到对应的connection，这样在服务结束的时候我们就能够close掉这个connection了。该hook会在如下几种ConnState状态的时候调用。

+ StateNew：新的连接，并且马上准备发送请求了
+ StateActive：表明一个connection已经接收到一个或者多个字节的请求数据，在server调用实际的handler之前调用hook。
+ StateIdle：表明一个connection已经处理完成一次请求，但因为是keepalived的，所以不会close，继续等待下一次请求。
+ StateHijacked：表明外部调用了hijack，最终状态。
+ StateClosed：表明connection已经结束掉了，最终状态。

通常，我们不会进入hijacked的状态（如果是websocket就得考虑了），所以一个可能的hook函数如下，参考[http://rcrowley.org/talks/gophercon-2014.html](http://rcrowley.org/talks/gophercon-2014.html)

```
s.ConnState = func(conn net.Conn, state http.ConnState) {
    switch state {
    case http.StateNew:
        // 新的连接，计数加1
        s.wg.Add(1)
    case http.StateActive:
        // 有新的请求，从idle conn pool中移除
        s.mu.Lock()
        delete(s.conns, conn.LocalAddr().String())
        s.mu.Unlock()
    case http.StateIdle:
        select {
        case <-s.quit:
            // 如果要关闭了，直接Close，否则加入idle conn pool中。
            conn.Close()
        default:
            s.mu.Lock()
            s.conns[conn.LocalAddr().String()] = conn
            s.mu.Unlock()
        }
    case http.StateHijacked, http.StateClosed:
        // conn已经closed了，计数减一
        s.wg.Done()
    }
```

当结束的时候，会走如下流程：

```
func (s *Server) Close() error {
    // close quit channel, 广播我要结束啦
    close(s.quit)
    
    // 关闭keepalived，请求返回的时候会带上Close header。客户端就知道要close掉connection了。
    s.SetKeepAlivesEnabled(false)
    s.mu.Lock()
    
    // close listenser
    if err := s.l.Close(); err != nil {
        return err 
    }
    
    //将当前idle的connections设置read timeout，便于后续关闭。
    t := time.Now().Add(100 * time.Millisecond)
    for _, c := range s.conns {
        c.SetReadDeadline(t)
    }
    s.conns = make(map[string]net.Conn)
    s.mu.Unlock()
    
    // 等待所有连接结束
    s.wg.Wait()
    return nil
}
```

好了，通过以上方法，我们终于能从容的关闭server了。但这里仅仅是针对跟客户端的连接，实际还有MySQL连接，Redis连接，打开的文件句柄，等等，总之，要实现优雅的服务关闭，真心不是一件很简单的事情。
