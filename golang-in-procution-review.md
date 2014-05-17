最近看了一篇关于go产品开发最佳实践的文章，[go-in-procution](http://peter.bourgon.org/go-in-production/)。作者总结了他们在用go开发过程中的很多实际经验，我们很多其实也用到了，鉴于此，这里就简单的写写读后感，后续我也争取能将这篇文章翻译出来。后面我用soundcloud来指代原作者。

## 开发环境

在soundcloud，每个人使用一个独立的GOPATH，并且在GOPATH直接按照go规定的代码路径方式clone代码。

    $ mkdir -p $GOPATH/src/github.com/soundcloud
    $ cd $GOPATH/src/github.com/soundcloud
    $ git clone git@github.com:soundcloud/roshi
    
对于go来说，通常的工程管理应该是如下的目录结构：

    proj/
        src/
            modulea/
                a.go
            moudleb/
                b.go
            app/
                main.go
        pkg/
        bin/
        
然后我们在GOPATH里面将proj的路径设置上去，这样就可以进行编译运行了。这本来没啥，但是如果我们要将其代码提交到github，并允许另外的开发者使用，我们就不能将整个proj的东西提交上面，如果提交了，就很蛋疼了。外面的开发者可能这么引用：

    import "github.com/yourname/proj/src/modulea"
    
但是我们自己在代码里面就可以直接：

    import "github.com/yourname/proj/modulea"
    
如果外面的开发者需要按照去掉src的引用方式，只能把GOPATH设置到proj目录，如果import的多了，会让人崩溃的。
    
我曾今也被这事情给折腾了好久，终于再看了vitess的代码之后，发现了上面这种方式，觉得非常不错。

## 工程目录结构

如果一个项目中文件数量不是很多，直接放在main包里面就行了，不需要在拆分成多个包了，譬如：

    github.com/soundcloud/simple/
        README.md
        Makefile
        main.go
        main_test.go
        support.go
        support_test.go

如果真的有公共的类库，在拆分成单独的包处理。

有时候，一个工程可能会包括多个二进制应用。譬如，一个job可能需要一个server，一个worker或者一个janitor，在这种情况下，建立多个子目录作为不同的main包，分别放置不同的二进制应用。同时使用另外的子目录实现公共的函数。

    github.com/soundcloud/complex/
    README.md
    Makefile
    complex-server/
        main.go
        main_test.go
        handlers.go
        handlers_test.go
    complex-worker/
        main.go
        main_test.go
        process.go
        process_test.go
    shared/
        foo.go
        foo_test.go
        bar.go
        bar_test.go

这点我的做法稍微有一点不一样，主要是参考vitess，我喜欢建立一个总的cmd目录，然后再在里面设置不同的子目录，这样外面就不需要猜测这个目录是库还是应用。

## 代码风格

代码风格这没啥好说的，直接使用gofmt解决，通常我们也约定gofmt的时候不带任何其他参数。

最好将你的编辑器配置成保存代码的时候自动进行gofmt处理。

Google最近发布了go的[代码规范](https://code.google.com/p/go-wiki/wiki/CodeReviewComments)，soundcloud做了一些改进：

- 避免命名函数返回值，除非能明确的表明含义。
- 尽量少用make和new，除非真有必要，或者预先知道需要分配的大小。
- 使用struct{}作为标记值，而不是bool或者interface{}。譬如set我们就用map[string]struct{}来实现，而不是map[string]bool。


如果一个函数有多个参数，并且单行长度很长，需要拆分，最好不用java的方式：

    // Don't do this.
    func process(dst io.Writer, readTimeout,
        writeTimeout time.Duration, allowInvalid bool,
        max int, src <-chan util.Job) {
        // ...
    }

而是使用：

    func process(
        dst io.Writer,
        readTimeout, writeTimeout time.Duration,
        allowInvalid bool,
        max int,
        src <-chan util.Job,
    ) {
    	// ...
    }


类似的，当构造一个对象的时候，最好在初始化的时候就传入相关参数，而不是在后面设置：

    f := foo.New(foo.Config{    
        Site: "zombo.com",            
        Out:  os.Stdout,
        Dest: conference.KeyPair{
            Key:   "gophercon",
            Value: 2014,
        },
    })
    
    // Don't do this.
    f := &Foo{} // or, even worse: new(Foo)
    f.Site = "zombo.com"
    f.Out = os.Stdout
    f.Dest.Key = "gophercon"
    f.Dest.Value = 2014
    
如果一些变量是后续通过其他操作才能获取的，我觉得就可以在后续设置了。

## 配置

soundcloud使用go的flag包来进行配置参数的传递，而不是通过配置文件或者环境变量。

flag的配置是在main函数里面定义的，而不是在全局范围内。

    func main() {
        var (
            payload = flag.String("payload", "abc", "payload data")
            delay   = flag.Duration("delay", 1*time.Second, "write delay")
        )
        flag.Parse()
        // ...
    }


关于使用flag作为配置参数的传递，我持保留意见。如果一个应用需要特别多的配置参数，使用flag比较让人蛋疼了。这时候，使用配置文件反而比较好，我个人倾向于使用json作为配置，原因在[这里](http://blog.csdn.net/siddontang/article/details/23595817)。

## 日志

soundcloud使用的是go的log日志，他们也说明了他们的log并不需要太多的其他功能，譬如log分级等。对于log，我参考python的log写了一个，在[这里](https://github.com/siddontang/golib/tree/master/log)。该log支持log级别，支持自定义loghandler。

soundcloud还提到了一个telemetry的概念，我真没好的办法进行翻译，据我的了解可能就是程序的信息收集，包括响应时间，QPS，内存运行错误等。

通常telemetry有两种方式，推和拉。

推模式就是主动的将信息发送给特定的外部系统，而拉模式则是将其写入到某一个地方，允许外部系统来获取该数据。

这两种方式都有不同的定位，如果需要及时，直观的看到数据，推模式是一个很好的选择，但是该模式可能会占用过多的资源，尤其是在数据量大的情况下面，会很消耗CPU和带宽。

soundcloud貌似采用的是拉模型。

关于这点我是深表赞同，我们有一个服务，需要将其信息发送到一个统计平台共后续的信息，开始的时候，我们使用推模式，每产生一条记录，我们直接通过http推给后面的统计平台，终于，随着压力的增大，整个统计平台被我们发挂了，拒绝服务。最终，我们采用了将数据写到本地，然后通过另一个程序拉取再发送的方式解决。

## 测试

soundcloud使用go的testing包进行测试，然后也使用flag的方式来进行集成测试，如下：

    // +build integration

    var fooAddr = flag.String(...)
    
    func TestToo(t *testing.T) {
        f, err := foo.Connect(*fooAddr)
        // ...
    }

因为go test也支持类似go build那种flag传递，它会默认合成一个main package，然后在里面进行flag parse处理。

这种方式我现在没有采用，我都是在测试用例里面直接写死了一个全局的配置，主要是为了方便的在根目录进行 go test ./...处理。不过使用flag的方式我觉得灵活性很大，后面如果有可能会考虑。

go的testing包提供的功能并不强，譬如没有提供assert_equal这类东西，但是我们可以通过reflect.DeepEqual来解决。

## 依赖管理

这块其实也是我非常想解决的。现在我们的代码就是很暴力的用go get来解决依赖问题，这个其实很有风险的，如果某一个依赖包更改了接口，那么我们go get的时候可能会出问题了。

soundcloud使用了一种vendor的方式进行依赖管理。其实很简单，就是把依赖的东西全部拷贝到自己的工程下面，当做自己的代码来使用。不过这个就需要定期的维护依赖包的更新了。

如果引入的是一个可执行包，在自己的工程目录下面建立一个\_vendor文件夹（这样go的相关tool例如go test就会忽略该文件夹的东西）。把\_vendor作为单独的GOPATH，例如，拷贝github.com/user/dep到\_vendor/src/github.com/user/dep下面。然后将\_vendor加入自己的GOPATH中，如下：

    GO ?= go
    GOPATH := $(CURDIR)/_vendor:$(GOPATH)
    
    all: build
    
    build:
        $(GO) build
        
如果引入的是一个库，那么将其放入vendor目录中，将vendor作为package的前缀，例如拷贝github.com/user/dep到vendor/user/dep，并更改所有的相关import语句。

因为我们并不需要频繁的对这些引入的工程进行go get更新处理，所以大多数时候这样做都很值。

我开始的时候也采用的是类似的做法，只不过我不叫vendor，而叫做3rd，后来为了方便还是决定改成直接go get，虽然知道这样风险比较大。没准后续使用godep可能是一个不错的解决办法。

## 构建和部署

soundcloud在开发过程中直接使用go build来构建系统，然后使用一个Makefile来处理正式的构建。

因为soundcloud主要部署很多无状态的服务，类似Heroku提供了很简单的一种方式：

    $ git push bazooka master
    $ bazooka scale -r <new> -n 4 ...
    $ # validate
    $ bazooka scale -r <old> -n 0 ...
    
这方面，我们直接使用一个简单的Makefile来构建系统，如下：

    all: build 
    
    build:
        go install ${SRC}
    
    clean:
        go clean -i ${SRC}
    
    test:
        go test ${SRC} 
    
应用程序的发布采用最原始的scp到目标机器在重启的方式，不过现在正在测试使用salt来发布应用。而对于应用程序的启动，停止这些，我们则使用supervisor来进行管理。


## 总结

总的来说，这篇文章很详细的讲解了用go进行产品开发过程中的很多经验，希望对大家有帮助。
