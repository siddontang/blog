# nginx虚拟主机解决企业内外网访问

在企业里面部署服务，需要面临的一个问题就是不同企业复杂的网络环境。通常来说，私有云只需要在企业内部使用，但是也有很多企业需要通过外网能访问。同时，对于不同网络的访问请求，系统也需要进行不同的处理。譬如内网用户请求下载直接可以rewrite到对应的内网下载机上，但外网用户请求下载则可能需要通过代理进行。

因为我们的系统使用nginx作为网络总的入口，所以，自然通过部署nginx来解决内外网的访问问题。对于私有云产品来说，内网的nginx server是很好配置的，难点在于如何配置外网的server，因为外网有很多种网络环境，需要分别考虑。

## 基础知识

在进行配置之前，首先列举一些nginx的配置需要了解的基本知识。


首先，来看一个最简单的nginx配置

    http {
        server {
            listen 192.168.1.10:80;
            server_name www.domain.com;
            location / {
                return 200 "Hello World";
            }
        }
    }

在上面这个例子中，nginx启动了一个server，该server监听192.168.1.10的80端口，server_name为 www.domain.com。

ip和port大家很好理解，对于server_name，可以认为就是发送http请求header里面的host。

当外部发送 http://192.168.1.10:80/hello 这个http请求的时候，监听80端口的这个server会处理。

同时，我们也可以通过域名来访问，如http://www.domain.com/hello，因为www.domain.com跟server_name配置一样，所以nginx也能处理响应。当然，前提是企业必须配置dns将该域名指定到192.168.1.10上面。

对于任何http请求，nginx都是首先获取一个匹配的server，而具体nginx选择哪一个server，则是如下流程：

- 通过listen的ip以及port确定server
- 如果有多个server，则通过server_name再次确定
- 如果仍然有多个，则按照配置顺序选择第一个

所以通过nginx来响应内外网的请求，也就是配置不同server的过程。

对于外网来说，通常来说有2种情况，具有独立的外网IP以及NAT映射的外网IP，如果提供了外网域名，通过域名解析的IP也仍然是上述两种情况。

## 独立外网IP

假设内网ip为192.168.1.10，实际的外网ip为10.20.189.217。

### 无域名

对于具有独立外网IP的机器来说，nginx很好配置。如下

    server {
        listen 192.168.1.10:80;
        server_name 192.168.1.10;
    }

    server {
        listen 10.20.189.217:80;
        server_name 10.20.189.217;
    }


可以看到，我们直接可以通过listen监听不同的ip来配置内外网server，

或者我们也可以通过如下方式：

    server {
        listen 80;
        server_name 192.168.1.10;
    }

    server {
        listen 80;
        server_name 10.20.189.217;
    }

这里，nginx监听同一个端口，通过对应的server_name来进行区分内外网。对于有独立外网IP的情况，建议使用前一种方式，直接listen ip:port。

### 有域名

如果有域名，那么我们可以在server_name中填入相应的域名信息。

    server {
        listen 192.168.1.10:80;
        server_name www.domain.com;
    }

    server {
        listen 10.20.189.217:80;
        server_name www.domain.com;
    }


可以看到，如果内外网都有相同的域名，那么listen的时候就必须得填入ip信息，如果只监听端口，nginx没法通过server_name区分内外网。

## NAT外网IP

如果外网IP为NAT映射的，那么nginx是不能直接listen这个IP的。假设NAT映射端口仍然为80，外网NAT地址为10.20.189.217。

### 无域名

    server {
        listen 80;
        server_name 192.168.1.10;
    }

    server {
        listen 80;
        server_name 10.20.189.217;
    }

可以看到，无域名情况比较简单，我们可以通过server_name来区分内外网。

### 有不同域名

如果内外网有不同域名，那么情况也跟无域名一样，通过配置server_name区分。

    server {
        listen 80;
        server_name www.domain1.com;
    }

    server {
        listen 80;
        server_name www.domain2.com;
    }

### 有相同域名

如果内外网有相同域名，那么就不能通过监听同一个端口来区分内外网了。笔者能想到的做法就是nginx监听不同的端口。

    server {
        listen 80;
        server_name www.domain1.com;
    }

    server {
        listen 10080;
        server_name www.domain2.com;
    }

这里，通过监听10080来响应外网的请求，这样NAT的映射端口就需要改变。


## end

可以看到，内外网的配置，其实就是nginx vhost的配置，而关键点就在于listen以及server_name。详细可以参考[How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)，[Server names](http://nginx.org/en/docs/http/server_names.html)。