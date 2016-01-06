## 起因

Docker算是现在非常火的一个项目，但笔者对其一直不怎么感冒，毕竟没啥使用场景。只是最近，笔者需要在自己的mac电脑上面安装项目的开发环境，发现需要安装MySQL，LedisDB，xcodis，Redis，Zookeeper等一堆东西，而同样的流程仍然要在Windows的机器上面再来一遍，陡然觉得必须得有一个更好的方式来管理整个项目的开发环境了。自然，笔者将目光放到了Docker上面。

根据官方自己的介绍，Docker其实是一个为开发和运维人员提供构建，分发以及运行分布式应用的开源平台（野心真的不小，难怪CoreOS要新弄一个Rocket来跟他竞争的）。

Docker主要包括Docker Engine，一个轻量级的运行和包管理工具，Docker Hub，一个用来共享和自动化工作流的云服务。实际在使用Docker的工程中，我们通常都是会在Docker Hub上面找到一个base image，编写Dockerfile，构建我们自己的image。所以很多时候，学习使用Docker，我们仅需要了解Docker Engine的东西就可以了。

至于为啥选用Docker，原因还是很明确的，轻量简单，相比于使用VM，Docker实在是太轻量了，笔者在自己的mac air上面同时可以运行多个Docker container进行开发工作，而这个对VM来说是不敢想象的。

后面，笔者将结合自己的经验，来说说如何构建一个MySQL Docker，以及当中踩过的坑。

## MySQL Docker

笔者一直从事MySQL相关工具的开发，对于MySQL的依赖很深，但每次安装MySQL其实是让笔者非常头疼的一件事情，不同平台安装方式不一样，加上一堆设置，很容易就把人搞晕了。所以自然，我的Docker第一次尝试就放到了MySQL上面。

对于mac用户，首先需要安装boot2docker这个工具才能使用Docker，这个工具是挺方便的，但也有点坑，后续会说明。

笔者前面说了，通常使用Docker的方式是在Hub上面找一个base image，虽然Hub上面有很多MySQL的image，但笔者因为开发[go-mysql](https://github.com/siddontang/go-mysql)，需要在MySQL启动的时候传入特定的参数，所以决定自行编写Dockerfile来构建。

首先，笔者使用的base image为ubuntu:14.04，Dockerfile文件很简单，如下:

```
FROM ubuntu:14.04

# 安装MySQL 5.6，因为笔者需要使用GTID
RUN apt-get update \
    && apt-get install -y mysql-server-5.6

# 清空apt-get的cache以及MySQL datadir
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/* /var/lib/mysql

# 使用精简配置，主要是为了省内存，笔者机器至少要跑6个MySQL
ADD my.cnf /etc/mysql/my.cnf

# 这里主要是给mysql_install_db脚本使用
ADD my-default.cnf /usr/share/mysql/my-default.cnf

# 增加启动脚本
ADD start.sh /start.sh
RUN chmod +x /start.sh

# 将MySQL datadir设置成可外部挂载
VOLUME ["/var/lib/mysql"]

# 导出3306端口
EXPOSE 3306

# 启动执行start.sh脚本
CMD ["/start.sh"]
```

我们需要注意，对于MySQL这种需要存储数据的服务来说，一定需要给datadir设置VOLUMN，这样你才能存储数据。笔者当初就忘记设置VOLUMN，结果启动6个MySQL Docker container之后，突然发现这几个MySQL使用的是同一份数据。

如果有VOLUMN, 我们可以在`docker run`的时候指定对应的外部挂载点，如果没有指定，Docker会在自己的vm目录下面生成一个唯一的挂载点，我们可以通过`docker inspect`命令详细了解每个container的情况。

对于`start.sh`，比较简单：

+ 判断MySQL datadir下面有没有数据，如果没有，调用`mysql_install_db`初始化。
+ 允许任意ip都能使用root账号访问，`mysql -uroot -e "GRANT ALL ON *.* TO 'root'@'%' IDENTIFIED BY '' WITH GRANT OPTION;"`，否则我们在外部无法连接MySQL。
+ 启动mysql

构建好了MySQL Docker image，我们就能使用`docker run`来运行了，很简单

```
docker run -d -p 3306:3306 --name=mysql siddontang/mysql:latest
```

这里，我们基于siddontang/mysql这个image创建了一个名叫mysql的container并运行，它会调用`start.sh`脚本来启动MySQL。

而我们通过`docker stop mysql`就可以停止mysql container了。

如果笔者需要运行多个MySQL，仅仅需要多新建几个container并运行就可以了，当然得指定对应的端口。可以看到，这种方式非常的简单，虽然使用`mysqld_multi`也能达到同样的效果，但是如果我需要在新增一个MySQL实例，`mysqld_mutli`还需要去更改配置文件，以及在对应的MySQL里面设置允许`mysqld_multi stop`的权限，其实算是比较麻烦的。而这些，在Docker里面，一个`docker run`就搞定了。

完整的构建代码在这里，[mysql-docker](https://github.com/siddontang/mysql-docker)，你也可以pull笔者提交到Hub的image `siddontang/mysql`来直接使用`docker pull siddontang/mysql:latest`。

## Boot2Docker Pitfall

从前面可以看到，Docker的使用是非常方便的，但笔者在使用的时候仍然碰到了一点坑，这里记录一下。

### IP

最开始碰到的就是ip问题，笔者在run的时候做了端口映射，但是外部使用MySQL客户端死活连接不上，而这个只在笔者mac上面出现，linux上面正常，后来发现是boot2docker的问题，我们需要使用`boot2docker ip`返回的ip来访问container，在笔者的机器上面，这个ip为192.168.59.103。

### Volumn

仍然是boot2docker的问题，笔者在`docker run`的时候，使用`-v`来将外部的目录绑定到datadir这个VOLUMN上面，这个在linux上面是成功的，可是在mac上面，笔者发现`mysql_install_db`死活没有权限写入磁盘。后来才知道，boot2docker只允许对自己VM下面的路径进行绑定。鉴于在mac下面仅仅是调试，数据不许持久化保存，这个问题也懒得管了。反正只要不删除掉container，数据还是会在的。

## Flatten Image

在使用Dockerfile构建自己的image的时候，对于Dockerfile里面的每一步，Docker都会生成一个layer来对应，也就是每一步都是一次提交，到最后你会发现，生成的image非常的庞大，而当你push这个image到Hub上面的时候，你的所有layer都会提交上去，加之我们国家的网速水平，会让人崩溃的。

所以我们需要精简生成的image大小，也就是flatten，这个Docker官方还没有支持，但至少我们还是有办法的：

+ `docker export` and `docker import`，通过对特定container的export和import操作，我们可以生成一个无历史的新container，详见[这里](http://tuhrig.de/flatten-a-docker-container-or-image/)。
+ [docker-squash](https://github.com/jwilder/docker-squash)，很方便的一个工具，笔者就使用这个进行image的flatten处理。

## 后记

总的来说，Docker还是很容易上手的，只要我们熟悉了它的命令，Dockerfile的编写以及相应的运行机制，就能很方便的用Docker来进行团队的持续集成开发。而在生产环境中使用Docker，笔者还没有相关的经验，没准后续私有云会采用Docker进行部署。

后续，对于多个Container的交互，以及服务发现，扩容等，笔者也还需要好好研究，CoreOS没准是一个方向，或者研究下[rocket](https://github.com/coreos/rocket) :-)