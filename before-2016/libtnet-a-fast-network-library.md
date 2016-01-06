# How do I develop Libtnet : a fast network library

Libtnet is a high performance library powered by C++ . It refers tornado in Python and supplies easy to use API, you can see more here: https://github.com/siddontang/libtnet.

## Why to develop?

There are many high performance network libraries, like libev, libevent, libuv, etc… But I still want to develop a new one, why? Maybe below reasons:

1, Learn lots of network programming knowledge.

2, Study how to design the architecture for a high performance library.

3, Improve my coding capability.

4, Challenge myself.

5, Maybe make me more famous. (Aha)

6, Most important, interesting.

As you see, many reasons let me to do it, so I begin my journey.

Below, I will list some key points about how do I choose, design, develop and test Libtnet, I hope some may help users who want to develop their own network libraries.

## Key Point

### Operating System

Sometimes, cross platform is evil, I can only develop Libtnet in my spare time and have no enough to support all popular OS, so I decide to support only Linux which is very suitable for developing high performance network library.

### Program Language

I use some program languages: C, C++, Python, Lua and Go, considering performance, c and c++ are still the fastest. I decide to use C++ only because I have been using it for eight years and have lots experience to make me develop quickly.

### Protocol

Libtnet only supports TCP, not supports UDP. UDP is fast but not reliable, also you can use some mechanisms to implement a reliable UDP, but I still think only TCP can solve many problems.

Libtnet also supports HTTP and Websocket which are widely used for web applications.

### I/O Strategy

In The C10K Problem, the author lists some i/o strategies to develop a high performance network library. Libtnet is Linux only, of course uses epoll to handle lots of connections.

Epoll has two mode: LT(level-triggered) and ET(edge-triggered). Libtnet uses LT because it can reduce the chance for errors. ET will only notice you only once if something occurs, if you forget to handle it, it will not tell you anymore. But LT will tell you all the time until you handle it.

Also some people think ET has a better performance, LT is not bad too.

Libtnet also supports prefork mode which is very suitable for stateless application like HTTP.

### Async

Libtnet is event-driven based on epoll, when a event comes, epoll will call the specialized callback function set before, like “onread”, “onwrite”, “onerror”, etc.

Writing asynchronous code in C++ is not an easy thing, because async may break code logic. Some people use coroutine(maybe using ucontext) to solve it, but I don’t use because it may cause heavily context switch(maybe I am wrong). Libtnet uses callback to handle async event.

### Callback

Using callback must care lifetime of the connection and the context associated with the connection.

Libtnet tries to solve it with smart pointer and std::function + std::bind.

In C++, using shared_ptr and weak_ptr is an easy way to handle lifetime. Use can assign a shared_ptr object to weak_ptr for outer use, if object was dead, weak_ptr can not successfully lock.

Some libraries like libev will let user set a context object for later callback use. I don’t like it, so Libtnet uses std::function instead, and you can use std::bind to set context with shared_ptr and the associated function.

### HTTP and Websocket

Libtnet uses http_parser to parse http request, http_parser is an awesome thing, very fast, easy to embed, I highly recommend it.

Libtnet only supports Websocket RFC6455, its format is easy so I write a parser by myself.

### TimingWheel

How to find a connection is dead, TCP is very reliable, but sometimes it may not detect remote was closed. TCP keepalive can slove it, but I still to use headbeat in Libtnet.

Libtnet uses timingwheel to handle connection’s keepalive, if some times later, a connection has read no data, Libtnet will consider it dead and close it directly.

### Test and Bench

I write some test to check Libtnet’s validation. Writing test is boring but the only way to solve whether it can be used now.

I build a comet-test to let Libtnet supports 1 million connections at same time and find that it can handle it easily.

I use ab to bench Libtnet and find that it has an amazing performance.

## At Last

Libtnet is my first fully open-source project, above is the summary for my hard work.

Libtnet does not make me more famous, but developing it make me learn lots of things, which is the most value for me.

I hope anyone can taste it, if you feel delicious, please tell me, I will be very happy.