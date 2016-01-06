# 在go中使用linked channels进行数据广播

原文在[这里](http://rogpeppe.wordpress.com/2009/12/01/concurrent-idioms-1-broadcasting-values-in-go-with-linked-channels/)（需翻墙），为啥想要翻译这篇文章是因为在实际中也碰到过如此的问题，而该文章的解决方式很巧妙，希望对大家有用。


在go中channels是一个很强大的东西，但是在处理某些事情上面还是有局限的。其中之一就是一对多的通信。channels在多个writer，一个reader的模型下面工作的很好，但是却不能很容易的处理多个reader等待获取一个writer发送的数据的情况。

处理这样的情况，可能的一个go api原型如下：

    type Broadcaster …
    
    func NewBroadcaster() Broadcaster
    func (b Broadcaster) Write(v interface{})
    func (b Broadcaster) Listen() chan interface{}
    
broadcast channel通过NewBroadcaster创建，通过Write函数进行数据广播。为了监听这个channel的信息，我们使用Listen，该函数返回一个新的channel去接受Write发送的数据。

这套解决方案需要一个中间process用来处理所有reader的注册。当调用Listen创建新的channel之后，该channel就被注册，通常该中间process的主循环如下：

    for {
        select {
            case v := <-inc:
                for _, c := range(listeners) {
                    c <- v
                }
            case c := <- registeryc:
                listeners.push(c)
        }
    }
    
这是一个通常的做法，（译者也经常这么干）。但是该process在处理数据广播的时候会阻塞，直到所有的readers读取到值。一个可选的解决方式就是reader的channel是有buffer缓冲的，缓冲大小我们可以按需调节。或者当buffer满的时候我们将数据丢弃。

但是这篇blog并不是介绍上面这套做法的。这篇blog主要提出了另一种实现方式用来实现writer永远不阻塞，一个慢的reader并不会因为writer发送数据太快而要考虑分配太大的buffer。

虽然这么做不会有太高的性能，但是我并不在意，因为我觉得它很酷。我相信我会找到一个很好的使用地方的。

首先是核心的东西：

    type broadcast struct {
        c chan broadcast
        v interface{}
    }
    
这就是我说的linked channel，但是其实是[Ouroboros data structure](http://wadler.blogspot.com/2009/11/list-is-odd-creature.html)。也就是，这个struct实例在发送到channel的时候包含了自己。

从另一方面来说，如果我有一个chan broadcast类型的数据，那么我就能从中读取一个broadcast b，b.v就是writer发送的任意数据，而b.c，这个原先的chan broadcast，则能够让我重复这个过程。

另一个可能让人困惑的地方在于一个带有缓冲区的channel能够被用来当做一个1对多广播的对象。如果我定义如下的buffered channel：

    var c = make(chan T, 1)
   
任何试图读取c的process都将阻塞直到有数据写入。

当我们想广播一个数据的时候，我们只是简单的将其写入这个channel，这个值只会被一个reader给获取，但是我们约定，只要读取到了数据，我们立刻将其再次放入该channel，如下：

    func wait(c chan T) T {
        v := <-c
        c <-v
        return v
    }
    
结合上面两个讨论的东西，我们就能够发现如果在broadcast struct里面的channel如果能够按照上面的方式进行处理，我们就能实现一个数据广播。

代码如下：

    package broadcast

    type broadcast struct {
        c   chan broadcast;
        v   interface{};
    }

    type Broadcaster struct {
        // private fields:
        Listenc chan chan (chan broadcast);
        Sendc   chan<- interface{};
    }

    type Receiver struct {
        // private fields:
        C chan broadcast;
    }

    // create a new broadcaster object.
    func NewBroadcaster() Broadcaster {
        listenc := make(chan (chan (chan broadcast)));
        sendc := make(chan interface{});
        go func() {
            currc := make(chan broadcast, 1);
            for {
                select {
                case v := <-sendc:
                    if v == nil {
                        currc <- broadcast{};
                        return;
                    }
                    c := make(chan broadcast, 1);
                    b := broadcast{c: c, v: v};
                    currc <- b;
                    currc = c;
                case r := <-listenc:
                    r <- currc
                }
            }
        }();
        return Broadcaster{
            Listenc: listenc,
            Sendc: sendc,
        };
    }

    // start listening to the broadcasts.
    func (b Broadcaster) Listen() Receiver {
        c := make(chan chan broadcast, 0);
        b.Listenc <- c;
        return Receiver{<-c};
    }

    // broadcast a value to all listeners.
    func (b Broadcaster) Write(v interface{})   { b.Sendc <- v }

    // read a value that has been broadcast,
    // waiting until one is available if necessary.
    func (r *Receiver) Read() interface{} {
        b := <-r.C;
        v := b.v;
        r.C <- b;
        r.C = b.c;
        return v;
    }
    
下面就是译者的东西了，这套方式实现的很巧妙，首先它解决了reader register以及unregister的问题。其次，我觉得它很好的使用了流式化处理的方式，当reader读取到了一个值，reader可以将其传递给下一个reader继续使用，同时自己在开始监听下一个新的值的到来。

译者自己的一个测试用例：

    func TestBroadcast(t *testing.T) {
        b := NewBroadcaster()

        r := b.Listen()

        b.Write("hello")

        if r.Read().(string) != "hello" {
            t.Fatal("error string")
        }

        r1 := b.Listen()

        b.Write(123)

        if r.Read().(int) != 123 {
            t.Fatal("error int")
        }

        if r1.Read().(int) != 123 {
            t.Fatal("error int")
        }

        b.Write(nil)

        if r.Read() != nil {
            t.Fatal("error nit")
        }

        if r1.Read() != nil {
            t.Fatal("error nit")
        }
    }
    
当然，这套方式还有点不足，主要就在于Receiver Read函数，并不能很好的与select进行整合，具体可以参考该作者另一篇blog [http://rogpeppe.wordpress.com/2010/01/04/select-functions-for-go/](http://rogpeppe.wordpress.com/2010/01/04/select-functions-for-go/)。
