# nginx性能优化

最近在测试服务器压力的时候，发现使用tornado的服务benchmark上不去，顶多1500左右，nginx即使开了8个进程，在响应请求的时候有一个work进程的cpu超高，达到100%的情况。
 
对于cpu超高的情况，当初我们都认为是2.6.18网卡中断只能在一个cpu上处理，导致cpu高，这虽然是一个原因，但是短期内升级整个系统是一个不太可能的事情。
 
鉴于官方说tornado性能很高，所以总觉得我们在某些地方使用有问题，看了nginx以及tornado的源码，发现有几个地方我们真没注意。
 
- listen backlog，nginx默认的backlog是511，而tornado则是100，对于这种设置，如果并发量太大，因为backlog不足会导致大量的丢包。
     
    将nginx以及tornado listen的时候backlog改大成20000，同时需要调整net.ipv4.tcp_max_syn_backlog，net.ipv4.tcp_timestamps，net.ipv4.tcp_tw_recycle等相关参数。
    
- accept_mutex，将其设置为off，nginx默认为on，是为了accept的解决惊群效应，但是鉴于nginx只有8个进程，同时并发量大，每个进程都唤醒都能被处理，所以关闭。
 
做了上面简单的两个操作之后，ab benchmark发现nginx的cpu负载比较平均，同时不会出现upstream request timeout以及cannot assign requested address等错误。
 
同时，直接压tornado也第一次达到了4000的rps，通过nginx proxy到tornado则在3200左右。

虽然只是修改了几个配置，性能就提升了很多，后续对于nginx，还有很多需要研究的东西。

版权声明：自由转载-非商用-非衍生-保持署名 [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)

 