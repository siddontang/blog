# mtunnel - a simple http tunnel

# 介绍

mtunnel是一个简易的http隧道工具，使用python实现，基于tornado。

为了解决远端用户服务器的一些问题，我们需要远程连接到用户的服务器上面，但因为企业安全性问题，外部根本不可能访问相关服务器，只能通过用户的本地机器去访问，但是用户本地机器也在内网环境下面，外部不能直接连接。

因为用户的机器在内网环境下面，也不可能通过通常的代理去连接。所以比较好的方式就是通过一台双方都能访问的公网服务器，来交换双方的数据。

mtunnel通过http tunnel的方式来进行数据的交互。它将双方交互的数据封装到http的body里面，通过http协议进行传输。这样有几个好处：

- 不用关心交互的具体协议，mtunnel只是将双方交互的所有信息完全封装到http body里面进行传递。
- http一般不会被企业的网络安全规则给屏蔽，内网穿透性更好。

假设，企业服务器启动了sshd，而我们这边使用putty。流程如下：

![image](https://raw.github.com/siddontang/blog/master/asserts/mtunnel-flow.png)

- putty直接将数据发送给forward proxy，由forward proxy将数据通过http body发送给server。
- server收到数据之后将其放置在一个buffer中。如果reverse proxy这时候已经连接到server，则直接将数据发送给reverse proxy。
- reverse proxy定时去server获取数据，如果有则将其发送给sshd。
- sshd返回的数据同样流程返回给putty。

## 使用

mtunnel分为3个部分，与putty交互的forward proxy，与sshd交互的reverse proxy以及负责双方数据中转的server。

### start server

        python server.py -p 8888

假设sever的ip地址为10.20.187.118，监听port为8888。

### start reverse proxy

因为我们需要通过用户本地机器去访问服务器，所以首先必须用户同意让我们访问，也就是他需要在本地启动reverse proxy。

        python rproxy.py -host 10.20.189.241 -p 22 --server 10.20.187.118:8888

host和port连接的是sshd的ip和port，而server则是服务器的地址。
reverse proxy如果成功连接上了server，则会获取一个channel id。后续所有的通信都必须通过该channel id进行。

### start forward proxy

        python fproxy.py --server 10.20.187.118:8888 --channel 112122 -p 8889
        
我们在本地启动forward proxy，监听本地8889端口，server为服务器的地址，而channel则是用户启动reverse proxy之后获取的channel id，由用户负责告诉我们。

### use

当做了上述操作之后，我们只需要启动putty，连接forward proxy。然后putty就能与远端的sshd交互了。


## 远程协助？

为什么不使用QQ远程协助这种类似功能？主要有以下几点原因：

- QQ远程协助能看到用户本地机器的很多信息，企业用户对于安全性问题比较敏感。
- 网络问题，现在很多企业的网络环境非常不好，我们就碰到过太多次远程的时候网络坑爹造成完全无法工作的情况。

## 后续方向

现在的版本只是一个最基本的版本，很多问题没有考虑，后续主要考虑以下几个方面：

- 安全性，尤其要保证channel id的安全性。后续考虑使用签名机制等。
- 网络容错，还没怎么很好的处理网络出问题的情况，譬如网络断开等。
- P2P，数据都走server会有延时，稳定等问题，如果能够P2P互通，就很强大了。

代码在[这里](http://https://github.com/siddontang/mtunnel)，该版本只是笔者使用python实现的一个非常基础的版本，很多地方存在不足，后续会慢慢完善。

版权声明：自由转载-非商用-非衍生-保持署名 [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)




