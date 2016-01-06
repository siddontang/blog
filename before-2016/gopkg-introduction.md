在使用go的过程中，我们有时候会引入一些第三方库来使用，而通常的方式就是使用```go get```，但是这种方式有一个很严重的问题，如果第三方库更新了相关接口，很有可能你就无法使用了，所以我们一套很好地包管理机制。

在[读生产环境下go语言最佳实践有感](http://blog.csdn.net/siddontang/article/details/25601611)一文中，我介绍过soundcloud公司的做法，直接将第三库的代码check下来，放到自己工程的vendor目录里面，或者使用godep。

不过现在，我发现了一种更好的包管理方式[gopkg](http://labix.org/gopkg.in)。它通过约定使用带有版本号的url来让go tool去check指定的版本库，虽然现在只支持github的go repositories，但是我觉得已经足够强大。

一个很简单的例子，我们通过如下方式获取go的yaml包

```
go get gopkg.in/yaml.v1
```

而实际上，该yaml包对应的地址为：

```
https://github.com/go-yaml/yaml
```

yaml.v1表明版本为v1，而在github上面，有一个对应的v1 branch。

gopkg支持的url格式很简单：

```
gopkg.in/pkg.v3      → github.com/go-pkg/pkg (branch/tag v3, v3.N, or v3.N.M)
gopkg.in/user/pkg.v3 → github.com/user/pkg   (branch/tag v3, v3.N, or v3.N.M)
```

我们使用v.N的方式来定义一个版本，然后再github上面对应的建立一个同名的分支。gopkg支持```(vMAJOR[.MINOR[.PATCH]])```这种类型的版本模式，如果存在多个major相同的版本，譬如v1，v1.0.1，v1.1.2，那么gopkg会选用最高级别的v1.1.2使用，譬如有如下版本：

+ v1
+ v2.0
+ v2.0.3
+ v2.1.2
+ v3
+ v3.0

那么gopkg对应选用的方式如下：

+ pkg.v1 -> v1
+ pkg.v2 -> v2.1.2
+ pkg.v3 -> v3.0

gopkg不建议使用v0，也就是0版本号。

gopkg同时列出了一些建议，在更新代码之后是否需要升级主版本或者不需要，一些必须升级主版本的情况：

+ 删除或者重命名了任何的导出接口，函数，变量等。
+ 给接口增加，删除或者重命名函数
+ 给函数或者接口增加参数
+ 更改函数或者接口的参数或者返回值类型
+ 更改函数或者接口的返回值个数
+ 更改结构体

而一下情况，则不需要升级主版本号：

+ 增加导出接口，函数或者变量
+ 给函数或者接口的参数名字重命名了
+ 更改结构体

上面都提到了更改结构体，譬如我给一个结构体增加字段，就可能不需要升级主版本，但是如果删除结构体的一个导出字段，那就必须要升级了。如果只是单纯的更改改结构体里面非导出字段的东西，也不需要升级。

更加详细的信息，请直接查看[gopkg](http://labix.org/gopkg.in)

可以看到，gopkg使用了一种很简单地方式让我们方便的对go pakcage进行版本管理。于是我也依葫芦画瓢，给我的log package做了一个v1版本的，你可以直接```go get gopkg.in/siddontang/go-log.v1/log```。