# 使用mio开发web framework - base

最近，笔者要用rust实现一个高性能网络服务，首先就需要选择一个好的异步网络库，在c++里面我们有太多选择，libev，libevent，libuv，甚至笔者自己也写过一个libtnet，不过在rust里面，mio几乎就是唯一的方案了。

mio实现很简单，就是在系统相关函数epoll，kqueue上面封装了自己的相关逻辑，抽象出好用的接口供外面使用。如果大家熟悉python tornado，就可以发现，mio就非常类似于tornado里面的IOLoop。

mio的使用很简单，我们只需要创建一个EventLoop，然后实现自己的Handler，就可以运行起来了。

mio的Handler是一个Trait，定义如下：

```rust
pub trait Handler: Sized {
    type Timeout;
    type Message;
    
    // 注册的token有相关事件的时候调用，譬如我们注册了一个socket，
    // socket可读或者可写的时候就会调用ready。
    fn ready(&mut self, event_loop: &mut EventLoop<Self>, token: Token, events: EventSet) {
    }

    // 有通知的时候调用。
    fn notify(&mut self, event_loop: &mut EventLoop<Self>, msg: Self::Message) {
    }

    // 注册的timer到期了调用
    fn timeout(&mut self, event_loop: &mut EventLoop<Self>, timeout: Self::Timeout) {
    }

    // 收到interupted signal的时候调用。
    fn interrupted(&mut self, event_loop: &mut EventLoop<Self>) {
    }

    // 每轮末尾的时候调用。
    fn tick(&mut self, event_loop: &mut EventLoop<Self>) {
    }
}
```

**一个简单的例子：**

```rust
use mio::{EventLoop, Handler};
struct BaseHandler;

impl Handler for BaseHandler {
    type Timeout = ();
    type Message = ();
}

let mut event_loop = EventLoop::<BaseHandler>::new().unwrap();
event_loop.shutdown();
```

在上面的例子中，我们实现了一个简单的BaseHandler，因为Handler这个Trait本身有默认的函数实现，所以BaseHandler没有干任何事情。

我们通过`EventLoop::new`创建一个event loop，但是没有干任何事情就shutdown了。

**Timeout的例子：**

```rust
struct TimeoutHandler;

impl Handler for TimeoutHandler {
    type Timeout = u32;
    type Message = ();

    fn timeout(&mut self, event_loop: &mut EventLoop<Self>, _: Self::Timeout) {
        event_loop.shutdown();
    }
}

let mut event_loop = EventLoop::new().unwrap();
event_loop.timeout_ms(100, 0).unwrap();
event_loop.run(&mut TimeoutHandler).unwrap();
```

上面的TimeoutHandler我们实现了timeout函数，里面直接shutdown这个event loop了，然后在run之前，我们通过timeout_ms这个函数，注册了一个timer，100ms之后过期，调用对应的timeout函数。

**Notify例子：**

```rust
struct NotifyHandler;

impl Handler for NotifyHandler {
    type Timeout = ();
    type Message = u32;

    fn notify(&mut self, event_loop: &mut EventLoop<Self>, _: Self::Message) {
        event_loop.shutdown();
    }
}

let mut event_loop = EventLoop::new().unwrap();
let sender = event_loop.channel();
let child = thread::spawn(move || {
   sender.send(0).unwrap();
});
event_loop.run(&mut NotifyHandler).unwrap();
child.join().unwrap();
```

上面的NotifyHandler实现了notify接口，在run之前，我们通过channel函数生成了一个sender，mio的EventLoop是一个单线程的loop，如果要在其他线程跟EventLoop进行交互，只能通过channel生成的sender。

上面的例子，我们生成了sender并且将其move到另一个新的线程中，然后在新的线程里面send了一个message，event loop会在自己的事件循环里面检测到这个事件，触发notify操作。

**Tick的例子：**

```rust
struct TickHandler;

impl Handler for TickHandler {
    type Timeout = ();
    type Message = ();

    fn tick(&mut self, event_loop: &mut EventLoop<Self>) {
        if !event_loop.is_running() {
            // Handle quit here.
            return;
        }
    }

    fn notify(&mut self, event_loop: &mut EventLoop<Self>, _: Self::Message) {
        event_loop.shutdown();
    }
}

let mut event_loop = EventLoop::new().unwrap();
event_loop.run(&mut TickHandler).unwrap();
```

EventLoop在每轮处理的最后，会调用tick这个回调函数，有时候我们会通过notify来告诉EventLoop结束，但是如果我们在notify这个message里面处理close，没准这个message后面还有一些message需要处理，没准这时候因为之前就已经在close里面释放相关资源了导致panic，所以笔者更倾向的一种做法就是close在tick里面处理，因为tick是每轮最后一个调用的函数，我们可以保证之后不会再处理任何的事件了。

**TODO：**

这里并没有说明ready的用法，因为这个跟socket关系比较大，后续笔者开始实现TcpServer的时候在详细说明。