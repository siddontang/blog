#lua与c的交互

关于lua和c的交互，主要有两个方面，一是lua调用c的函数，而另一个则是c调用lua函数。而这些都是通过lua stack来进行的。

##c调用lua

在c里面使用lua，主要是通过lua_call这类函数，下面来自lua manual的例子：

    lua_getglobal(L, "f");                  /* function to be called */
    lua_pushstring(L, "how");                        /* 1st argument */
    lua_getglobal(L, "t");                    /* table to be indexed */
    lua_getfield(L, -1, "x");        /* push result of t.x (2nd arg) */
    lua_remove(L, -2);                  /* remove 't' from the stack */
    lua_pushinteger(L, 14);                          /* 3rd argument */
    lua_call(L, 3, 1);     /* call 'f' with 3 arguments and 1 result */
    lua_setglobal(L, "a");                         /* set global 'a' */

该例子等同于直接在lua里面调用 a = f("how", t.x, 14)。
通过上面的例子可以看到，在c中使用lua是一件很容易的事情，首先获取需要调用的lua函数，然后将其需要的参数依次压入stack，然后通过lua_call调用，该函数调用的返回值也压入stack，供c去获取。

##lua调用c

对于lua调用c，我们首先需要将c函数注册给lua，而注册给lua的函数，需要满足 int (\*lua_CFunction)(lua_State\* pState) 这种类型。如下例子:

    int multi(lua_State* pState)
    {
        int data1 = int(lua_tonumber(pState, 1));
        int data2 = int(lua_tonumber(pState, 2));
        lua_pushnumber(pState, data1 * data2);
        return 1;
    }

    lua_register(pState, multi, "multi");

    #for use in lua
    #a = multi(10, 20)

我们通过lua_register将mutli函数注册给lua，该函数接受两个参数，并且有一个返回值。当lua调用multi的时候，会将参数压入stack，所以我们可以通过lua_tonumber(pState, 1)和lua_tonumber(pState, 2)来获取，其中1为第一个参数10，2为第二个参数20。当运行完成之后，multi函数通过lua_pushnumber将结果压入lua堆栈，并通过return 1告知lua有一个返回值，为200。

可以看到，在lua中使用c也是一件很简单的事情。

#lua mainly or c mainly

通过上面的例子可以看出，lua与c是很方便的交互的，但是在实际的游戏项目中，我们首先必须确定的一个问题就是，代码逻辑是以lua为主还是以c为主。

- lua为主的游戏就是逻辑主要由lua负责，核心的对性能要求较高的逻辑则由c负责，游戏中的数据大多由lua负责。
- c为主的游戏则是逻辑主要由c负责，lua只是负责简单的配置。

这两种方式都有优劣，对于实际游戏项目来说，个人认为，应该采用lua为主，c作为高性能核心的方式。之所以这样选择，是因为游戏的逻辑变动很大，我们需要快速的进行代码迭代，这个对于lua来说非常方便。而对于核心引擎，因为变化不大，同时对性能要求较高，所以采用c是一个很好的选择。

#reg helper

在实际的游戏项目中，我们会遇到这样一个问题，假设c提供的函数为 int func(int a, int b)，如果这个函数要提供给lua使用，我们需要写一个对应的注册函数，如下:

    int func_wrapper(lua_State* pState)
    {
        int a = int(lua_tonumber(pState, 1));
        int b = int(lua_tonumber(pState, 2));
        int ret = func(a, b);
        lua_pushnumber(pState, ret);
        return 1;
    }

    lua_register(pState, func_wrapper, "func");

对于任意的c函数，我们需要写一个对应的wrapper用来注册给lua。如果项目中只有几个c函数，那么无所谓，但是如果需要注册给lua的c函数很多，那么对于每一个c函数写一个wrapper，是一件很不现实的事情。并且如果c函数的参数或者返回值有变化，我们同时需要修改对应的wrapper函数。基于上述原因，我们需要一套自动机制，能够将任意的c函数注册给lua使用。实际来说，我们需要提供一个函数，对于任意的c函数func，我们只需要调用register(func, "func")，那么就能直接注册给lua使用。

##traits

首先，我们必须面对的问题就是，不同的c函数，参数和返回值是不一样的，譬如对于int类型的参数，我们需要通过lua_tonumber获取数据，而对于const char\* 类型的参数，我们需要通过lua_tostring来获取，同理对于返回值也一样。所以我们需要一套机制，根据c函数不同的参数和返回值类型来调用lua对应的stack操纵函数。我们可以通过c++ traits来实现。

我们提供如下一套函数:
    
    template<typename T>
    struct TypeHelper{};
    bool getValue(TypeHelper<bool>, lua_State* pState, int index)
    {
        return lua_toboolean(pState, index) == 1;
    }
    char getValue(TypeHelper<char>, lua_State* pState, int index)
    {
        return static_cast<char>(lua_tonumber(pState, index));
    }
    int getValue(TypeHelper<int>, lua_State* pState, int index)
    {
        return static_cast<int>(lua_tonumber(pState, index));
    }
    void pushValue(lua_State* pState, bool value)
    {
        lua_pushboolean(pState, int(value));
    }

    void pushValue(lua_State* pState, char value)
    {
        lua_pushnumber(pState, value);
    }

    void pushValue(lua_State* pState, int value)
    {
        lua_pushnumber(pState, value);
    }

通过traits技术，对于不同的参数类型，我们可以调用对应的getValue函数，来从lua获取实际的数据，而对于返回值，通过c++自动的参数匹配，就能调用对应的pushValue函数。

##CCallHelper

上面解决了类型匹配的问题，下面就需要提供wrapper函数，用以封装c函数。这里我们提供call helper类来实现。

    template<typename Ret>
    class CCallHelper
    {
    public:
        static int call(Ret (*func)(), lua_State* pState)
        {
            Ret ret = (*func)();
            pushValue(pState, ret);
            return 1;
        }

        template<typename P1>
        static int call(Ret (*func)(P1), lua_State* pState)
        {
            P1 p1 = getValue(TypeHelper<P1>(), pState, 1);
            Ret ret = (*func)(p1);
            pushValue(pState, ret);
            return 1;
        }
    };

CCallHelper提供了静态的call函数，第一个参数就是实际c函数，通过函数模板可以进行任意c函数的匹配，这里只提供了匹配无参数和一个参数类型的c函数模板，我们可以扩展到支持任意参数个数，但也别太多了。因为对于任意c函数来说，可能有一个返回值，也可能没有返回值，那么我们如何匹配没有返回值的c函数呢？这里就是为什么我们需要CCallHelper的原因。在c++中，是不支持函数级别的模板特化的，但是类却可以，所以我们通过特化CCallHelper来匹配无返回值的c函数。如下：

    template<>
    class CCallHelper<void>
    {
    public:
        static int call(void (*func)(), lua_State* pState)
        {
            (*func)();
            return 0;
        } 

        template<typename P1>
        static int call(void (*func)(P1), lua_State* pState)
        {
            P1 p1 = getValue(TypeHelper<P1>(), pState, 1);
            (*func)(p1);
            return 0;
        }
    };

##CCallDispatcher

通过CCallHelper::call(func, pState)，我们就可以与lua进行交互，那么又如何调用到相应的CCallHelper呢？这里我们通过CCallDispatcher来进行，如下：

    template<typename Func>
    class CCallDispatcher
    {
    public:
        template<typename Ret>
        static int dispatch(Ret (*func)(), lua_State* pState)
        {
            return CCallHelper<Ret>::call(func, pState);
        }

        template<typename Ret, typename P1>
        static int dispatch(Ret (*func)(P1), lua_State* pState)
        {
            return CCallHelper<Ret>::call(func, pState);
        }
    };

通过CCallDispatcher，我们就可以将不同的c函数dispatch到不同的CCallHelper上面。

##CCallRegister and regFunction

解决了c函数派发调用的问题，最后我们就需要处理如何将任意的c函数注册给lua，代码如下：

    template<typename Func>
    class CCallRegister
    {
    public:
        static int call(lua_State* pState)
        {
            Func* func = static_cast<Func*>(lua_touserdata(pState, lua_upvalueindex(1));
            return CCallDispatcher<Func>::dispatch(*func, pState);
        }
    };

    template<typename Func>
    void regFunction(lua_State* pState, Func func, const char* funcName)
    {
        int funcSize = sizeof(Func);
        void* data = lua_newuserdata(pState, funcSize);
        memcpy(data, &func, funcSize);

        lua_pushcclosure(pState, CCallRegister<Func>::call, 1);
        lua_setglobal(pState, funcName);
    }

首先，我们提供CCallRegister类，里面提供了一个static的call函数，该函数满足lua注册格式，所以实际我们是将该函数注册给lua，在call函数里面，我们通过lua_touserdata(pState, lua_upvalueindex(1))来获取实际的func，然后传递给CCallDispatcher进行派发。而将call注册则是通过regFunction，该函数将实际的c函数func存储在一个userdata中，然后将该userdata绑定到对应的CCallRgister call上面，作为一个upvalue，这样当在lua里面调用call函数的时候，通过lua_upvalueindex获取对应的upvalue，则可以取到实际的c函数。

#一些设计上面的考虑

上述reghelper的实现，我已经放到github [luahelper](https://github.com/siddontang/luahelper)上面，并且在max os，gcc 4.2，lua5.2环境下面测试通过。

这里谈一些设计上面的问题，首先，我说的任意c函数，参数和返回值只能是基本数值类型，如bool，char，short，int，long，float，double以及字符串类型char\*等，这里我并没有提供复杂类型譬如class，struct的支持。之所以这样考虑，是因为我想保证lua与
c交互的简单，云风曾经说过[lua不是c++](http://blog.codingnow.com/2008/08/lua_is_not_c_plus_plus.html)，我本人当年也曾经用了2年的时间做了同样的事情。但是实现了这套东西，功能是狠强大了，以至于可以把lua当成c++来用了，但是这真的是使用lua正确的方式吗？我现在觉得，引入复杂的结合层，反而在某些时候会带来更大的复杂性，导致语言侧重点的混淆。所以有时候，我反而觉得，对于这种语言的交互，可能使用其他方式，譬如json，反而来的更容易。

#写在后面的话

对于lua和游戏开发的一些东西，其实一直想写，但是以前因为很多方面的原因而中途放弃。现在重新开始，有几个方面的原因，一个在于仍然对于游戏开发的热爱，做了4年的游戏开发，虽然现在从事云存储方面的研究，但是对游戏热情依旧。另一个方面在于lua5.2的发布，觉得是应该对以前游戏人生做一个总结了。

如果需要更强大的lua与c的交互，我觉得[swig](http://www.swig.org/)可能更合适。
