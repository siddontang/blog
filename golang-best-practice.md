在SoundCloud，我们组织我们的产品作为一个API为多个客户端提供服务。也就是说，我们的网站，移动客户端，移动应用，都是头等的客户端，使用同一个主API。在这个API之后，是很多的服务集合：SoundCloud其实就是一个面向服务的构架模式。

我们的是一个多语言的团队，也就是说我们使用很多种语言。但是我们多个服务（包括底层基础部件）都是采用go编写。事实上，我们已经再生产环境中使用go已经超过了两年半的时间。现在，一些我们用go实现的项目包括：


- Bazooka，我们内部的服务平台，比较类似Heroku或者Flynn。

- perimeter traffic tier采用了标准的nginx，HAProxy等，但是使用go服务进行协调。

- 我们的audio assets是基于S3的，但是上传，转码以及链接生成都是使用go。

- Search采用的是Elasticsearch，同时Explore采用的是复杂的机器学习模块，但是他们都是通过我们go基础模块进行整合的。

- Prometheus，一个早期的遥测系统，完全使用go编写。这个现在通过Cassandra支持，但是我们扩展了一个纯Go的扩展版本。

- 我们也用go实验实现了一个HTTP流式直播。

- 当然还包括了很多其他小的面向服务的产品

这些项目通过大概半打项目组编写，包括超过一打的独立SoundCloud gohers，他们大部分是go的全职开发者。在经历了这些日子，这些项目，以及在一堆各种各样的工程师中，我们对于go的开发，积累了很多经验，而这些对于其它想开始进行go开发的团队来说可能会很有帮助。

# 开发环境

On our laptops, we’ve got a single, global GOPATH. Personally, I like $HOME, but many others use a subdirectory under $HOME. We clone our repos into their canonical paths within the GOPATH, and work there directly. That is,

在我们的开发机器上面，我们每个人使用一个单独的，全局的GOPATH。我喜欢 $HOME，但是其他很多人使用$HOME的子目录。我们直接在GOPATH里面clone工程，并且在里面直接使用，如下：

    $ mkdir -p $GOPATH/src/github.com/soundcloud
    $ cd $GOPATH/src/github.com/soundcloud
    $ git clone git@github.com:soundcloud/roshi

在很早的时候，大多数我们一直在为这个惯例抗争，为了保持我们自己的代码惯例方式。这其实真没必要争论的。

For editors, many of use use vim, with various plugins. (I understand vim-go is a good one.) Many, including myself, use Sublime Text with GoSublime. A few use emacs. Nobody uses an IDE. I’m not sure that’s a best practice, it’s just interesting to note.

对于编辑器，多数人使用配置了很多插件的vim（我知道vim-go是一个很好的东西）。也有多数，包括我，使用Sublime，当然装上了GoSublime。一些很好的人使用emacs。没有人使用IDE。我并不确定这是一个最好的实践，只是觉得有意思记录一下。

# 代码结构

我们最好的实践是保持事情的简单。很多很多服务只有少量源文件的我们都直接放在main package里面。

    github.com/soundcloud/simple/
        README.md
        Makefile
        main.go
        main_test.go
        support.go
        support_test.go

例如，我们的搜索派发器，在两年后仍然保持这样。不要创建另外的目录结构除非你明确知道需要这样干。

有可能一些时候你需要创建一个新的支持类库。在你的工程下面新建一个子目录，并且按照全路径进行应用。如果这个新的包只有少量文件，或者就一个结构体，其实并没有需要将其拆分。

Sometimes a repo needs to contain multiple binaries; for example, when a job needs a server, a worker, and maybe a janitor. In those circumstances, put each binary in a separate package main in a separate subdirectory, and use other subdirectories (other packages) to implement shared functionality.

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

Note that there’s never a src directory involved. With the exception of a vendor subdirectory (more on that below) your repos shouldn’t contain a directory named src, or represent their own GOPATH.

这里并没有src目录。除开下面会说明的vendor子目录，你的工程里面并不应该包含src目录。

# 格式化和样式

首先也是最重要的，配置你的编辑器使得在你保存代码的时候能够进行go fmt（或者 goimports）。代码没有进行格式化处理的不允许提交。

我们以前使用一个很漂亮的广泛样式规范，但是Google最近发布了他们的代码规范文档，这跟我们的差不了多少，所以我们开始使用。

我们实际上约定了更多：

- 避免命名的函数返回值，除非他们明显能够增加清晰性。
- 避免make和new，除非是必要的（new(int)或者make(chan int)），或者我们预先知道需要分配的大小（make(map[int]string), n]，或者make([]int, 0, 256)）。
- 使用struct{}作为标记值，而不用bool或者interface{}。例如，set就是map[string]struct{}。一个信号channel就是chan struct{}，它发送一个明确不带任何信息的信号。

将过长的参数行拆分，不同于Java-style

    // Don't do this.
    func process(dst io.Writer, readTimeout,
        writeTimeout time.Duration, allowInvalid bool,
        max int, src <-chan util.Job) {
        // ...
    }

使用：

    func process(
        dst io.Writer,
        readTimeout, writeTimeout time.Duration,
        allowInvalid bool,
        max int,
        src <-chan util.Job,
    ) {
    	// ...
    }

类似的，当构造一个对象的时候：

    f := foo.New(foo.Config{    
        Site: "zombo.com",         Out:  os.Stdout,         Dest: conference.KeyPair{             Key:   "gophercon",
            Value: 2014,
        },
    })
    
也就是，在对象初始化的时候传入相关参数，是好于后续在设置它们的。

    // Don't do this.
    f := &Foo{} // or, even worse: new(Foo)
    f.Site = "zombo.com"
    f.Out = os.Stdout
    f.Dest.Key = "gophercon"
    f.Dest.Value = 2014

# 配置

我们尝试了很多方法给go程序传递配置：传递配置文件，从os.Getenv，直接获取环境变量，各种各样的value-add标志解析包。最后，最好的方式就是使用默认的flag包。严格的类型和简单的语义就足够我们需求了。

我们发布了主要的12-Factor应用，该应用通过环境变量传递配置，但是后来，我们通过一个启动脚本将环境变量转换成了flag。在程序和执行环境之间，Flags提供了一个明显的，完全文档化的表现层。（Flags act as an explicit and fully-documented surface area between a program and its operating context. ）

对于理解和执行应用，flags是无价的。

一个好的使用flag的习惯就是在main函数里面定义他们。这能阻止你从全局地方读取，并强迫你去忍受严格的为了让测试更容易的依赖注入。

    func main() {
        var (
            payload = flag.String("payload", "abc", "payload data")
            delay   = flag.Duration("delay", 1*time.Second, "write delay")
        )
        flag.Parse()
        // ...
    }

# 日志和遥测（telemetry）

我们使用了多个日志框架，他们有些提供了不同日志级别，输出路由，特定的格式化等功能。但是最终，我们使用了原始的log包，因为我们仅仅需要记录操作记录。也就是说，panic-level的错误需要被人工记录，而结构化的数据则会被其它机器消费。例如，search dispatcher发送任何它处理的包含上下文环境的请求，因此我们的分析器就能知道从新西兰来的IP有多频繁搜索了Lorde，或者其它。

任何从当前运行进程里面发送的东西我们都称之为遥测。包括请求相应时间，QPS，运行时错误，队列长度等。遥测通常有两种模式，推和拉。

推意味着数据发送给一个特定的外部系统。例如，Graphite，Statsd，AirBrake都使用的这种方式。

拉意味着在一些地方暴漏这些数据，并且允许外部的系统去获取他们。例如，expvar，Prometheus都是使用这种方式。（可能还有其他的？）

两种模式都有他们的定位。开始的时候，推能够直观和明确的使用，但是当数据量增长的时候，推需要占用更多的资源。当数据量越多，推消耗的越多，包括CPU和带宽。我们发现过去一个特定大小的基础构件，拉是应对扩展的唯一模式。在一个运行的系统中，有很多数据需要关注，所以，最好的实践：expvar or expvar-style metrics exposition。

# 测试和验证

我们尝试过多个不同的测试库以及框架，但是很快就放弃他们了。现在，我们使用的是最简单的testing包，通过数据驱动（或者表驱动）测试。我们对这个测试包没有啥抱怨的，除了他们没提供强大的值检测。不过通过reflect.DeepEqual能帮你简化，它能做到任意数据的比较。（例如expected vs got）

Package testing is geared around unit testing, but for integration tests, things are a little trickier. The process of spinning up external services typically depends on your integration environment, but we did find one nice idiom to integrate against them. Write an integration_test.go, and give it a build tag of integration. Define (global) flags for things like service addresses and connect strings, and use them in your tests.

// +build integration

var fooAddr = flag.String(...)

func TestToo(t *testing.T) {
	f, err := foo.Connect(*fooAddr)
	// ...
}
go test takes build tags just like go build, so you can call go test -tags=integration. It also synthesizes a package main which calls flag.Parse, so any flags declared and visible will be processed and available to your tests.

By validation, I mean static code validation. And fortunately Go has several great tools. I find it useful to consider the stages of writing code, when considering which tools to use.

When you do this	Run this
Save	go fmt (or goimports)
Build	go vet, golint, and maybe go test
Deploy	go test -tags=integration
Interlude

So far, nothing too crazy. When doing research to compile this list, what was notable to me was just how… uninteresting the conclusions were. Boring. I want to emphasize that these very lightweight, pure-stdlib conventions really do scale to large groups of developers and diverse project ecosystems. You absolutely don’t need your own error checking framework, or testing library, or flag parser, simply because your code base has grown beyond a certain size. Or you believe it might grow beyond a certain size! You truly ain’t gonna need it. The standard idioms and practices continue to function beautifully at scale.


Dependency management ∞

Dependency management! Whee! ᕕ( ᐛ )ᕗ

The state of dependency management in the Go ecosystem is a topic of hot debate, and we’ve definitely not figured out the perfect solution yet. However, we have settled on what seems like a good compromise.

How important is your project?	Your dependency management solution is…
Eh…	go get -d and hope!
Very.	VENDOR
(It’s worth noting that a shocking number of our long-term production services still rely on option 1. Still, because we don’t generally use a lot of third-party code, and because major problems are usually detected at build time, we can mostly get away with it.)

Vendoring means copying your dependencies into your project’s repo, and then using them when building. Depending on what you’re shipping, there are two best practice ways of vendoring.

Shipping a	Vendor directory name	Procedure
Binary	_vendor	Blessed build with prefixed GOPATH
Library	vendor	Rewrite your imports
If you ship a binary, create a _vendor subdirectory in the root of your repository. (With a leading underscore, so the go tool ignores it when doing e.g. go test ./....) Treat it like its own GOPATH; for example, copy the dependency github.com/user/dep to _vendor/src/github.com/user/dep. Then, write a so-called blessed build procedure, which prepends _vendor to any existing GOPATH. (Remember: GOPATH is actually a list of paths, searched in order by the go tool when resolving imports.) For example, you might have a top-level Makefile that looks like this:

GO ?= go
GOPATH := $(CURDIR)/_vendor:$(GOPATH)

all: build

build:
	$(GO) build
If you’re shipping a library, create a vendor subdirectory in the root of your repository. Treat it just like a prefix in package paths; for example, copy the dependency github.com/user/dep to vendor/user/dep. Then, rewrite all of your imports, transitively. While a pain, this seems to be the best available way to ensure actually-reproducible builds while remaining go get compliant. It’s worth noting that we don’t actually ship any libraries, so this method may be too much of a hassle in practice to be worthwhile.

How to actually copy the dependencies into your repository is another hot topic. The simplest way is to manually copy the files from a clone, which may be the best answer if you’re not concerned with pushing changes upstream. Some people use git submodules, but we found them very counterintuitive and difficult to manage. (So do many other people, for the record.) We’ve had good success with git subtrees, which appear to work like submodules ought to. And there’s plenty of work on tools to handle this work automatically. Right now, it looks like godep is the most actively developed, and is certainly worth investigating.


Building and deploying ∞

Building and deploying is tricky, because it’s so tightly coupled to your operational environment. I’ll describe our situation, because I think it’s a really good model, but it may not apply directly to other organizations.

For building, we generally use plain go build for development, and a Makefile for cutting official builds. That’s mostly because we’re polyglot, and our tooling needs to use a lowest common denominator. Also, our build system starts with a bare environment, and we need to bring our own compiler. (Our Makefiles are ugly!)

For deploying, the key abstraction for us is stateless versus stateful.

Mode	Example	Model	Deploying is called	Deploy in
Stateless	Request router	12-Factor	Scaling	Containers
Stateful	Redis	None, really	Provisioning	Containers?
We primarily deploy stateless services, in a manner very similar to Heroku.

$ git push bazooka master
$ bazooka scale -r <new> -n 4 ...
$ # validate
$ bazooka scale -r <old> -n 0 ...

Conclusions ∞

I intend this to be a kind of experience report from a large organization that’s been running Go in production for a relatively long time. While these are informed opinions, they’re still just opinions, so please take everything with a grain of salt. That said, Go’s greatest strength is its structural simplicity. The ultimate best practice is to embrace simplicity, rather than trying to circumvent it.

