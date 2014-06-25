作为一个Celery使用重度用户，看到[Celery Best Practices](https://denibertovic.com/posts/celery-best-practices/)这篇文章，不由得菊花一紧。干脆翻译出来，同时也会加入我们项目中celery的实战经验。

通常在使用Django的时候，你可能需要执行一些长时间的后台任务，没准你可能需要使用一些能排序的任务队列，那么Celery将会是一个非常好的选择。

当把Celery作为一个任务队列用于很多项目中后，作者积累了一些最佳实践方式，譬如如何用合适的方式使用Celery，以及一些Celery提供的但是还未充分使用的特性。

## 1，不要使用数据库作为你的AMQP Broker

数据库并不是天生设计成能用于AMQP broker的，在生产环境下，它很有可能在某时候当机（PS，当掉这点我觉得任何系统都不能保证不当吧！！！）。

作者猜想为啥很多人使用数据库作为broker主要是因为他们已经有一个数据库用来给web app提供数据存储了，于是干脆直接拿来使用，设置成Celery的broker是很容易的，并且不需要再安装其他组件（譬如RabbitMQ）。

假设有如下场景：你有4个后端workers去获取并处理放入到数据库里面的任务，这意味着你有4个进程为了获取最新任务，需要频繁地去轮询数据库，没准每个worker同时还有多个自己的并发线程在干这事情。

某一天，你发现因为太多的任务产生，4个worker不够用了，处理任务的速度已经大大落后于生产任务的速度，于是你不停去增加worker的数量。突然，你的数据库因为大量进程轮询任务而变得响应缓慢，磁盘IO一直处于高峰值状态，你的web应用也开始受到影响。这一切，都因为workers在不停地对数据库进行DDOS。

而当你使用一个合适的AMQP（譬如RabbitMQ）的时候，这一切都不会发生，以RabbitMQ为例，首先，它将任务队列放到内存里面，你不需要去访问硬盘。其次，consumers（也就是上面的worker）并不需要频繁地去轮询因为RabbitMQ能将新的任务推送给consumers。当然，如果RabbitMQ真出现问题了，至少也不会影响到你的web应用。

这也就是作者说的不用数据库作为broker的原因，而且很多地方都提供了编译好的RabbitMQ镜像，你都能直接使用，譬如[这些](https://registry.hub.docker.com/search?q=rabbitmq)。

对于这点，我是深表赞同的。我们系统大量使用Celery处理异步任务，大概平均一天几百万的异步任务，以前我们使用的mysql，然后总会出现任务处理延时太严重的问题，即使增加了worker也不管用。于是我们使用了redis，性能提升了很多。至于为啥使用mysql很慢，我们没去深究，没准也还真出现了DDOS的问题。

## 2，使用更多的queue（不要只用默认的）

Celery非常容易设置，通常它会使用默认的queue用来存放任务（除非你显示指定其他queue）。通常写法如下：

```
@app.task()
def my_taskA(a, b, c):
    print("doing something here...")

@app.task()
def my_taskB(x, y):
    print("doing something here...")
```

这两个任务都会在同一个queue里面执行，这样写其实很有吸引力的，因为你只需要使用一个decorator就能实现一个异步任务。作者关心的是taskA和taskB没准是完全两个不同的东西，或者一个可能比另一个更加重要，那么为什么要把它们放到一个篮子里面呢？（鸡蛋都不能放到一个篮子里面，是吧！）没准taskB其实不怎么重要，但是量太多，以至于重要的taskA反而不能快速地被worker进行处理。增加workers也解决不了这个问题，因为taskA和taskB仍然在一个queue里面执行。

## 3，使用具有优先级的workers

为了解决2里面出现的问题，我们需要让taskA在一个队列Q1，而taskB在另一个队列Q2执行。同时指定**x** workers去处理队列Q1的任务，然后使用其它的workers去处理队列Q2的任务。使用这种方式，taskB能够获得足够的workers去处理，同时一些优先级workers也能很好地处理taskA而不需要进行长时间的等待。

首先手动定义queue

```
CELERY_QUEUES = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('for_task_A', Exchange('for_task_A'), routing_key='for_task_A'),
    Queue('for_task_B', Exchange('for_task_B'), routing_key='for_task_B'),
)
```

然后定义routes用来决定不同的任务去哪一个queue

```
CELERY_ROUTES = {
    'my_taskA': {'queue': 'for_task_A', 'routing_key': 'for_task_A'},
    'my_taskB': {'queue': 'for_task_B', 'routing_key': 'for_task_B'},
}
```

最后再为每个task启动不同的workers
```
celery worker -E -l INFO -n workerA -Q for_task_A
celery worker -E -l INFO -n workerB -Q for_task_B
```

在我们项目中，会涉及到大量文件转换问题，有大量小于1mb的文件转换，同时也有少量将近20mb的文件转换，小文件转换的优先级是最高的，同时不用占用很多时间，但大文件的转换很耗时。如果将转换任务放到一个队列里面，那么很有可能因为出现转换大文件，导致耗时太严重造成小文件转换延时的问题。

所以我们按照文件大小设置了3个优先队列，并且每个队列设置了不同的workers，很好地解决了我们文件转换的问题。

## 4，使用Celery的错误处理机制

大多数任务并没有使用错误处理，如果任务失败，那就失败了。在一些情况下这很不错，但是作者见到的多数失败任务都是去调用第三方API然后出现了网络错误，或者资源不可用这些错误，而对于这些错误，最简单的方式就是重试一下，也许就是第三方API临时服务或者网络出现问题，没准马上就好了，那么为什么不试着重试一下呢？

```
@app.task(bind=True, default_retry_delay=300, max_retries=5)
def my_task_A():
    try:
        print("doing stuff here...")
    except SomeNetworkException as e:
        print("maybe do some clenup here....")
        self.retry(e)
```

作者喜欢给每一个任务定义一个等待多久重试的时间，以及最大的重试次数。当然还有更详细的参数设置，自己看文档去。

对于错误处理，我们因为使用场景特殊，例如一个文件转换失败，那么无论多少次重试都会失败，所以没有加入重试机制。

## 5，使用Flower

[Flower](http://celery.readthedocs.org/en/latest/userguide/monitoring.html#flower-real-time-celery-web-monitor)是一个非常强大的工具，用来监控celery的tasks和works。

这玩意我们也没怎么使用，因为多数时候我们都是直接连接redis去查看celery相关情况了。貌似挺傻逼的对不，尤其是celery在redis里面存放的数据并不能方便的取出来。

## 6，没事别太关注任务退出状态

一个任务状态就是该任务结束的时候成功还是失败信息，没准在一些统计场合，这很有用。但我们需要知道，任务退出的状态并不是该任务执行的结果，该任务执行的一些结果因为会对程序有影响，通常会被写入数据库（例如更新一个用户的朋友列表）。

作者见过的多数项目都将任务结束的状态存放到sqlite或者自己的数据库，但是存这些真有必要吗，没准可能影响到你的web服务的，所以作者通常设置```CELERY_IGNORE_RESULT = True```去丢弃。

对于我们来说，因为是异步任务，知道任务执行完成之后的状态真没啥用，所以果断丢弃。

## 7，不要给任务传递 Database/ORM 对象

这个其实就是不要传递Database对象（例如一个用户的实例）给任务，因为没准序列化之后的数据已经是过期的数据了。所以最好还是直接传递一个user id，然后在任务执行的时候实时的从数据库获取。

对于这个，我们也是如此，给任务只传递相关id数据，譬如文件转换的时候，我们只会传递文件的id，而其它文件信息的获取我们都是直接通过该id从数据库里面取得。

## 最后

后面就是我们自己的感触了，上面作者提到的Celery的使用，真的可以算是很好地实践方式，至少现在我们的Celery没出过太大的问题，当然小坑还是有的。至于RabbitMQ，这玩意我们是真没用过，效果怎么样不知道，至少比mysql好用吧。

最后，附上作者的一个Celery Talk [https://denibertovic.com/talks/celery-best-practices/](https://denibertovic.com/talks/celery-best-practices/)。