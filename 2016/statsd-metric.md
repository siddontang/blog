# StatsD Metric

## Why StatsD

在很多系统中，大家都能看到metric的踪影，我们通过监控metric的变化，就可能知道当前系统运行的状况。

Metric的方案有很多，譬如著名的[prometheus](https://prometheus.io/)，[statsd](https://github.com/etsy/statsd)等，也可以自己造轮子，毕竟通用的metric types也就那么几种，用好了足够用来监控系统了。

[Etcd](https://github.com/coreos/etcd)使用的是prometheus，看名字就知道很是cool的一个系统，笔者之前使用Etcd的时候碰到了一个超时问题，通过Etcd的metric发现是当前磁盘IO负载太高，使得Etcd的fsync太慢，从而导致请求超时的。

因为metric很重要，所以我们也决定在项目中引入metric。最开始，我们想的是直接使用memory的metric解决方案，但Etcd的团队推荐我们使用prometheus，可是这玩意并没有rust的client，于是我们就选择了另一个流行的解决方案StatsD。主要几个原因：

+ 协议简单外面可以非常方便的对接使用，rust也有相关的library。
+ 使用UDP，速度快，client这边即使频繁发送，也不会降低系统性能。
+ StatsD还支持多种backend，我们可以将StatsD收集到的信息转发到其他的系统譬如graphite，influxdb，prometheus上面。

## Usage StatsD

StatsD的使用非常简单，因为是node.js的，所以我们需要先安装好node环境，然后写好一个配置文件，直接启动就可以了，一个简单的配置文件：

```
{
  port: 8125
, backends: [ "./backends/console" ]
, console: { prettyprint: true }
}
```
这里，我们使用默认的8125 UDP端口，backend使用的是console，也就是StatsD会将收集到的metrics汇总输出到console上面，既然是console，那就prettyprint一下，好看一点 :-)

启动好StatsD之后，我们就可以通过nc简单使用了:

```
echo "foo:1|c" | nc -w 1 -u 127.0.0.1 8125
```

上面的例子中，我们发送了一个counter，metric的名字是foo，StatsD收到这条metric之后，会查看当前是不是已经有该foo的metric，并将对应的值加1，如果没有，则默认从0开始。

可以看到，metric的协议格式是非常简单的，如下：

```
<metricname>:<value>|<type>
```

也就是对于一个metric来说，我们只要想好他的名字以及对应的类型，然后发实际的数据给StatsD就可以了。

## Metric Types

### Counting

最简单的metric应该就是counter，也就是通常的计数功能，StatsD会将收到的counter value累加，然后在flush的时候输出，并且重新清零。所以我们用counter就能非常方便的查看一段时间某个操作的频率，譬如对于一个HTTP服务来说，我们可以使用counter来统计request的次数，finish这个request的次数以及fail的次数。

### Gauges

不同于Counter，Gauge在下次flush的时候是不会清零的，另外，gauge通常是在client进行统计好在发给StatsD的，譬如, `capacity:100|g` 这样的gauge，即使我们发送多次，在StatsD里面，也只会保存100，不会学counter那样进行累加。

但我们可以通过显示的加入符号来让StatsD帮我们进行累加，譬如:

```
capacity:+100|g
capacity:-100|g
```

假设我们原来的capacity gauge的值为100，经过上面的操作之后，gauge仍然是100。

如果我们需要记录当前的总用户数，或者CPU，Memory的usage，使用gauge就是一个不错的选择。

### Sets

Set用来计算某个metric unique事件的个数，譬如对于一个接口，可能我们想知道有多少个user访问了，我们可以这样:

```
request:1|s
request:2|s
request:1|s
```

StatsD就会展示这个request metric只有1，2两个用户访问了。

### Timing

最后再来说timing，timing顾名思义，就是记录某个操作的耗时，譬如：

```
foo:100|ms
```

上面的例子中，完成foo这个操作花费了100ms，但仅仅是记录这个操作的耗时，并不能让我们很好的知道当前系统的情况，所以通常，timing都是跟histogram一起来使用的。

在StatsD里面，配置histogram很简单，例如：

```
histogram: [ { metric: '', bins: [10, 100, 1000, 'inf']} ]
```

在上面的例子中，我们开启了histogram，这个histogram的bin的间隔是[-inf, 10ms)，[10ms - 100ms), [100ms - 1000ms), 以及[1000ms, +inf)，如果一个timing落在了某个bin里面，相应的bin的计数就加1，譬如：

```
foo:1|ms
foo:100|ms
foo:1|ms
foo:1000|ms
```

那么StatsD在console就会显示:

```
histogram: { bin_10: 2, bin_100: 0, bin_1000: 1, bin_inf: 1 } } },
```

## Summary

通过上面的例子可以看到，StatsD还是非常容易使用的，所以剩下的就是我们在代码里面根据实际情况加上metric了，但这里还有几点需要注意：

+ UDP虽然很快，但仍然可能会因为发送buffer满block当前进程，建议设置成noblock，对于metric来说，其实我们并不在意丢了几个包。
+ 埋点是一个辛苦活，太多或者太少的metric其实都没啥用。
+ metric也并不是万能的，它只是一个系统的汇总统计，有时候我们还需要借助log，flamegraph等其他方式来进行系统问题排查。

