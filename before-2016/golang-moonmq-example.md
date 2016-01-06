在上一篇moonmq的介绍中（[这里](http://blog.csdn.net/siddontang/article/details/22889495)），我仅仅简短的罗列了一些moonmq的设计想法，但是对于如何使用并没有详细说明，公司同事无法很好的使用。

对于moonmq的使用，其实很简单，样例代码在[这里](https://github.com/siddontang/moonmq/tree/master/example)，我们只需要处理好broker，consumer以及publisher的关系就可以了。

首先，我们需要启动一个broker，因为moonmq现在只支持tcp的自定义协议，所以broker启动的时候需要指定一个listen address。

    #启动broker
    ./simple_broker -addr=127.0.0.1:11182
    
启动了broker之后，我们就可以向该broker发送消息

    #向test这个queue发送 hello msg
    ./simple_publisher -addr=127.0.0.1:11182 -queue=test -msg=hello
    
然后在另一个shell里面接收消息
    
    #接收test这个queue的消息
    ./simple_consumer -addr=127.0.0.1:11182 -queue=test
    
    #output get msg: hello
    
如果没有消息，那么consumer就会一直等待，直到接收到消息。

这里详细说一下consumer的实现，

- 创建一个与broker的连接
        
        //create a client for use
        client := NewClient(config)
        
        //get a usable connection
        conn, _ := client.Get()

- 绑定queue
        
        //bind a queue
        //queue name : test
        //routingKey : ""
        //noAck : true
        ch, _ := conn.Bind("test", "", true)

- 接收消息

        //receive msg, block to wait until a msg received
        msg := ch.GetMsg()
        println(msg)

- 回执消息

        //if channel noAck is false, we must ack
        ch.Ack()

从上面的例子可以看出，使用moonmq很方便，后续我准备加入http的支持，使其更容易使用。

moonmq的代码在这里[https://github.com/siddontang/moonmq](https://github.com/siddontang/moonmq)。