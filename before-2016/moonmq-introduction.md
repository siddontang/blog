# 介绍

moonmq是一个用go实现的高性能消息队列系统，后续准备用于我们消息推送服务以及各个后台的异步任务。

在设计上面，moonmq主要借鉴了rabbitmq以及rocketmq相关的思想，但是做了很多减法，毕竟我不是要设计成一个非常通用的mq。

# 名词解释

- publisher，消息生产者
- consumer，消息消费者
- broker，消息中转站
- queue，消息存储队列

publisher给一个命名的queue发送消息msg，broker负责将msg存放在queue里面。

consumer可以关注自己感兴趣的queue，这样当queue里面有消息的时候，broker就会将该消息推送给该consumer。

# 推拉模型

在rocketmq里面，支持的是pull msg，而rabbitmq则是支持push和pull msg。moonmq只支持push msg。主要有如下考量：

- 当consumer在线的时候，push是最及时的，因为这时候铁定能把msg push成功。
- 当consumer离线，broker会保存离线消息，当consumer上线之后，broker仍然按照push的方式将离线消息进行推送。

另外，因为moonmq后续会支持我们的消息推送系统，如果采用pull模型，几十万的consumer定期的pull，我有点担心moonmq会吃不消。

# 消息类型

moonmq将msg分为direct和fanout，fanout就是广播消息，moonmq会将任何订阅了该queue的consumer进行msg push。

如果msg的type为direct，moonmq将会采用轮询的方式，选择一个consumer进行msg push。

# 消息优先级

moonmq不支持消息优先级，处理起来会很麻烦，而且通常我们并不需要特别精细的优先级控制。

可以通过一个简单的方式实现粗粒度的优先级控制：

- 设置queue1，queue2，queue3三个队列，queue1用来处理保存优先级最高的消息，queue2次之，queue3最低
- publisher发送消息的时候根据优先级发送到指定的queue上面去
- 我们可以有多个consumer处理queue1的消息，譬如3个，然后用2个处理queue2的，1个处理queue1的，这样实现优先级控制。

# 消息过滤

moonmq通过routing key来进行消息过滤。

publisher在给特定queue发送msg的时候，还可以指定对应的routing key，只有关注了该queue同时也指定了相同的routing key的consumer才会收到该msg。


# Ack

moonmq支持ack机制，当push一个msg到consumer的时候，consumer必须回应一个ack，moonmq才认为msg push成功。如果长时间没有ack，则moonmq会重新选择一个consumer再次发送。

ack能够很大程度的保证消息推送的成功率，但是对于消息的快速推送会有影响，所以moonmq也支持no ack模式，这种模式下moonmq只要发送成功了msg，就认为push成功，无需等待ack的回执。

# 延迟消息

这个现在还没支持，后续是情况而定

# 定时消息

难度比较大，不会实现

# 协议

moonmq采用的是类似rocketmq的协议，如下：

    |total length(4 bytes)|header length(4 bytes)|header json|body|

    total length = 4 + len(header json) + len(body)
    header length = len(header json)

在moonmq里面，我们使用Proto来定义协议

    type Proto struct {
        Method uint32 `json:"method"`
        
        Fields map[string]string `json:"fields"`
        
        Body []byte `json:"-"`
    }
   
moonmq的任何协议，都需要带上method，我们通过method进行实际的消息处理。

moonmq的method参考rabbitmq，有如下几种类型的method：

- 同步request method，客户端在发送request method之后必须等待对应的response method，在等待的过程中也能够处理push，error等异步method。
- 同步response method，对应特定的request method。
- 异步method，发送之后无需等待。

现阶段，moonmq支持如下同步method：

- auth, auth_ok
- publish, publish_o
- bind, bind_ok
- unbind, unbind_ok

同时支持如下异步method:

- push
- error
- heartbeat
- ack

# 后续

这只是moonmq的一个简单介绍，后续我们会不断完善moonmq，争取也能成为一个不错的mq产品。

moonmq的代码在这里[https://github.com/siddontang/moonmq](https://github.com/siddontang/moonmq)，期待大家的反馈。