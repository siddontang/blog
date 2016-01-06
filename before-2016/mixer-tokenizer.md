# 介绍

mixer希望在proxy这层就提供自定义路由，sql黑名单，防止sql注入攻击等功能，而这些的基石就在于将用户发上来的sql语句进行解析。也就是我最头大的词法分析和语法分析。

到现在为止，我只是实现了一个比较简单的词法分析器，用以将sql语句分解成多个token。而对于从token在进行语法分析，构建sql的AST，我现在还真没啥经验（编译原理太差了），急需牛人帮忙。

所以，这里只是简单介绍一下mixer的词法分析。

# tokenize

在很多地方，我们都需要进行词法分析，通常会有几种方式：

- 使用一个强大的工具，譬如lex，mysql-proxy就用的这种方式
- 使用正则表达式
- state machine

对于使用工具，我觉得有一个不怎么好的地方在于学习成本，譬如我用lex的时候就需要学习它的语法，同时通过工具生成的代码可读性都不怎么好，代码量大，更严重的是可能会比较慢。所以mysql自身也是自己实现一个词法分析模块。

而对于正则表达式，性能问题可能是一个很需要考虑的，而且复杂度并不比使用类似lex这样的工具低。

状态机可能是我觉得自己动手实现词法解析一个很好的方式，对于sql的词法解析，我觉得使用state machine的方式来自己写一个难度并不大，所以mixer自己实现了一个。

# state machine

通常，一个状态机的实现采用的是state + action + switch的做法，可能如下：

    switch state {
        case state1:
            state = action1()
        case state2:
            state = action2()
        case state3:
            state = action3()
    }
    
对于一个state，我们通过switch知道它将会由哪一个action进行处理，而对于每一个action，我们则知道执行完成之后下一个state是什么。

对于上面的实现，如果state过多，可能会导致太多的case语句，我们可以通过state function进行简化。

一个state function就是执行当前的state action，并且直接返回下一个state function。

我们可以这样做：

    type stateFn func(*Lexer) stateFn

    for state := startState; state != nil {
        state = state(lexer)
    }
    
所以我们需要实现的就是每一个state function以及对应的它的下一个需要执行的state function。

# mixer lexer

mixer的词法分析实现主要参考[这个](http://cuddle.googlecode.com/hg/talk/lex.html)。主要实现在[parser模块](https://github.com/siddontang/mixer/tree/master/src/parser)。

对于一个lexer，需要提供的是NextToken的功能，供外部获取下一个token，从而进行后续的操作（譬如语法分析）。

lexer的next token如下：

    func (l *Lexer) NextToken() (Token, error) {
        for {
            select {
                case t := <-l.tokens:
                    return t, nil
                default:
                    if l.state == nil {
                        return Token{TK_EOF, ""}, l.err
                    }
                    l.state = l.state(l)
                    if l.err != nil {
                        return Token{TK_UNKNOWN, ""}, l.err
                    }
            }
        }
    }
    
tokens是一个channel，每次state解析的token都会emit到这个channel上面，供NextToken获取，如果channel为空了，则再次调用state function。

可以看到，用go实现一个词法解析是很容易的事情，剩下的就是写相应的state function用来解析sql。

# todo

mixer的词法分析还有很多不完善的地方，譬如对于科学计数法数值的解析就不完善，后续准备参考mysql官方的词法分析模块在好好完善一下。

mixer的代码在这里[https://github.com/siddontang/mixer](https://github.com/siddontang/mixer)，希望感兴趣的童鞋共同完善。
