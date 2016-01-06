在go自带的timer实现中，采用的是通常的最小堆的方式，具体可以参见[这里](http://golang.org/src/pkg/runtime/time.goc)。

最小堆能够提供很好的定时精度，但是，在实际情况中，我们并不需要这样高精度的定时器，譬如对于一个连接，如果它在2分钟以内没有数据交互，我们就将其删除，2分钟并不需要那么精确，多几秒少几秒都无所谓的。

以前我们单独实现了一个[timingwheel](https://github.com/siddontang/golib/tree/master/timingwheel)，采用的是channel close的方式来处理低精度，超大量timer定时的问题，详见[这里](http://blog.csdn.net/siddontang/article/details/18370541)。

但是timingwheel只有After接口，远远不能满足实际的需求，于是我按照linux timer的实现方式，依葫芦画瓢，弄了一个go版本的实现。linux timer的实现，参考[这篇](http://blog.csdn.net/tianmohust/article/details/8707162)。

后续用go timer来表示我自己实现的timer。

在linux中，我们使用tick来表示一次中断的时间，用jiffies来表示系统自启动以来流逝的tick次数。在go timer中，我们在创建一个wheel的时候，需要指定一次tick的时间，如下：

    func NewWheel(tick time.Duration) *Wheel
    
Wheel是go timer统一对timer进行管理的地方。对于每一次tick，我们采用go自带的ticker进行模拟。

为了便于外部使用，我仍然提供的是跟go自己timer一样的接口，譬如：

    func NewTimer(d time.Duration) *Timer
    
在NewTimer中，参数d是一个time duration，我们还需要根据tick来进行换算，得到go timer中实际的expires，也就是在多少次jiffies后该timer触发。

譬如，NewTimer参数为10s，tick为1s，那么经过10个jiffies之后，该timer就会超时触发。如果tick为500ms，那么需要经过20个jiffies之后，该timer才会被触发。

所以timer超时jiffies的计算如下：

    expires = wheel.jiffies + d / wheel.tick
        
详细的代码在[https://github.com/siddontang/golib/tree/master/timer](https://github.com/siddontang/golib/tree/master/timer)。

