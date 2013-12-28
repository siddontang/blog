#introduction

工欲善其事，必先利其器。lua作为一门动态语言，虽然我已经习惯了使用print来进行代码调试，但是还是有很多童鞋觉得一款好用的调试器能更好的进行lua代码编写。所以在接手游戏的lua结合层之后，自然就需要提供一个debuger工具了。

我们只需要的是一个能快速进行lua代码调试的工具，所以不需要gdb那种额外复杂的功能，只需要提供几种简单的功能就行了，如下：

- c/continue   继续执行
- bt/backtrace 列出当前堆栈
- f/frame n    跳转到frame n
- l/list b e   列出源代码，b为起始行，e为结束行
- p/print v    打印v的值
- n/next       执行，跳过下一行，包括跳过子函数
- s/step       执行，直到碰到不同的一行
- return       执行，直到该函数结束

虽然调试器实现的功能很简单，但是对于大多数应用来说，已经完全足够使用。

#lua debug library

lua提供了一个debug library，我们就通过这个库来实现一个调试器。
首先，我们需要注册一个lua debug hook，并且绑定LUA_MASKLINE，LUA_MASKRET，LUA_MASKCALL事件，这样当lua代码执行的时候如果碰到相应的事件，则会调用我们注册的debug hook。

当debug hook调用的时候，程序就进入debug模式，这时候就可以输入对应的命令进行执行。

#db

只要注册了debug hook，那么每次lua代码执行的时候碰到对应的事件就会调用注册hook，如果每次调用都进入debug模式，那是很影响程序运行的，所以我们需要一种机制，只在需要的时候进入debug模式。

在gdb里面，我们通过设置断点来进入可调式模式，虽然在lua里面也可以这么做，但这里我们采用了一种更简单的方法。我们给lua注册一个db函数，当lua执行到db函数的时候，程序才会进入debug模式。因为lua是动态语言，如果我们需要在另一个地方也进行调试，只需要再次加入db函数，重启程序即可。

所以这里我们debug hook函数内部实现是一个状态机，当没有进入db的时候，虽然lua也会进行调用该hook，但该hook内部不作任何处理。只有当执行db函数进入debug模式，hook内部才会有相应处理。

我们在debug hook里面提供了多种状态，包括none hook，step hook，next hook和return hook。

- none hook，没有进入debug模式，该hook不做任何处理
- step hook，进入debug的step模式，当lua代码执行到新的一行代码时候做处理
- next hook，进入debug的next模式，当lua代码跳过下一行时候做处理
- return hook，进入debug的return模式，当lua执行到当前函数退出时候做处理

#continue

continue会让程序继续执行。该命令会让hook切入none hook状态，直到下次lua执行db函数进入debug模式。

#step

step会让hook切入step hook状态，该hook会监听LUA_MASKLINE事件，当该事件发生时候，step hook进行处理，打印当前代码，并再次进入debug模式，供下次命令输入。

#next

next会让hook切入next hook状态，该hook也会监听LUA_MASKLINE事件，但是next跟step最大的区别在于next是跳过一行，也就是说如果执行的lua代码下一行是一个函数调用，step会进入函数内部，而next则会执行该函数，并跳过该函数这一行直到下一行。

所以next需要进行判断LUA_MASKLINE是否进入了一个新的函数，这里我们通过函数堆栈深度来进行，当lua代码执行到一个新的函数的时候，它的函数堆栈深度会加1，所以我们只需要记录当前的堆栈深度a，next执行到下一次LUA_MASKLINE时候，获取堆栈深度b，如果a小于b，那么表明进入了一个新的函数，所以我们不需要处理，直到再次获取的堆栈深度等于a。

#return

return会让hook切入return hook状态，该hook会监听LUA_MASKRET事件，当该事件发生，return hook进行处理。这里我们仍然需要进入是否进入新函数调用的判断，因为我们只想监听的是当前函数的LUA_MASKRET事件，所以我们仍需要像next那样进行堆栈深度的判断。

#backtrace/frame/list

backtrace，frame，list这几个命令这里列在一起，是因为他们都跟lua_getstack，lua_getinfo这两个函数有关系。我们通过lua_getstack初始化指定栈帧的lua_Debug结构，然后在通过调用lua_getinfo获取相关栈帧信息。

#print value

print value命令可能算是最复杂的一个命令，因为有多个逻辑处理。当我们通过frame定位到某一层栈帧之后，就可以通过print打印相关的对象数据，供调试使用。

当print value的时候，首先我们查找value是否在当前函数里面local变量里面，如果没有则查找该函数的upvalue，如果仍然没有，则查找global，如果都没找到，则输出nil。

#code

代码在[luahelper](https://github.com/siddontang/luahelper)的debughelper，只是一个简单的实现，还有一些问题需要考虑。

因为lua的debug hook注册的时候只能提供一个hook，所以为了简单起见，我对DebugHelper使用了单例模式，但是最好是一个lua实例对应一个debuger。要做到这样，自己想到了两种可能方法:

- 使用LUAI_EXTRASPACE，并通过luai_userstateopen，luai_userstateclose将debuger绑定到lua实例上面，并通过luai_userstatethread进行debuger在coroutine的迁移。这种方法需要重新编译lua代码，适合集成lua源码的项目。
- debuger内部使用一个map来进行对应，但是在lua里面需要替换coroutine的创建，因为创建的coroutine也需要对应到同一个debuger上面。

两种方法都懒得弄了，以后有机会去尝试一下。

#end

这里只是简单的实现了一个lua debuger，但是功能我觉得足以可以在实际项目中应用了。只是越来觉得，对于动态语言，print和log才是我最喜欢的代码调试方式，因为简单而且强迫你去思考整个程序的运行流程。不过把debuger放在这里，也算是对自己以前游戏开发的一个总结吧。
