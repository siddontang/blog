# Ticker

最近的项目用go实现的服务器需要挂载大量的socket连接。如何判断连接是否还存活就是我们需要考虑的一个问题了。

通常情况下面，socket如果被客户端正常close，服务器是能检测到的，但是如果客户端突然拔掉网线，或者是断电，那么socket的状态在服务器看来可能仍然是established。而实际上该socket已经不可用了。

为了判断连接是否可用，通常我们会用timer机制来定时检测，在go里面，这非常容易实现，如下：

    ticker := time.NewTicker(60 * time.Second)
    
    for {
        select {
            case <-ticker.C:
                if err := ping(); err != nil {
                    close()
                }
        }
    }
    
上面我们使用一个60s的ticker，定时去ping，如果ping失败了，证明连接已经断开了，这时候就需要close了。

这套机制比较简单，也运行的很好，直到我们的服务器连上了10w+的连接。因为每一个连接都有一个ticker，所以同时会有大量的ticker运行，cpu一直在30%左右徘徊，性能不能让人接受。

其实，我们只需要的是一套高效的超时通知机制。

# Close channel to broadcast

在go里面，channel是一个很不错的东西，我们可以通过close channel来进行broadcast。如下：

    ch := make(bool)
    
    for i := 0; i < 10; i++ {
        go func() {
            println("begin")
            <-ch
            println("end")
        }
    }
    
    time.Sleep(10 * time.Second)
    
    close(ch)

上面，我们启动了10个goroutine，它们都会因为等待ch的数据而block，10s之后close这个channel，那么所有等待该channel的goroutine就会继续往下执行。

# TimingWheel

通过channel这种close broadcast机制，我们可以非常方便的实现一个timer，timer有一个channel ch，所有需要在某一个时间 “T” 收到通知的goroutine都可以尝试读该ch，当T到达时候，close该ch，那么所有的goroutine都能收到该事件了。

timingwheel的使用很简单，首先我们创建一个wheel

    //这里我们创建了一个timingwheel，精度是1s，最大的超时等待时间为3600s
    w := timingwheel.NewTimingWheel(1 * time.Second, 3600)
    
    //等待10s
    <-w.After(10 * time.Second)
    
因为timingwheel只有一个1s的ticker，并且只创建了3600个channel，系统开销很小。当我们程序换上timingwheel之后，10w+连接cpu开销在10%以下，达到了优化效果。

timingwheel的代码在[这里](https://github.com/siddontang/golib/tree/master/timingwheel)。