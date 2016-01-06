在go里面，虽然有log模块，但是该模块提供的功能并不强，譬如就没有我们常用的level log功能，但是自己实现一个log模块也并不困难。

对于log的level，我们定义如下:

    const (
        LevelTrace = iota
        LevelDebug
        LevelInfo
        LevelWarn
        LevelError
        LevelFatal
    )    
    
相应的，提供如下几个函数:

    func Trace(format string, v ...interface{}) 
    func Debug(format string, v ...interface{}) 
    func Info(format string, v ...interface{}) 
    func Warn(format string, v ...interface{}) 
    func Error(format string, v ...interface{}) 
    func Fatal(format string, v ...interface{}) 

另外，对于一个log，我们可能将其写入不同的地方，譬如可能直接输出到stdout，或者写文件，或者写socket，在创建特定logger的时候，我们需要指定对应的handler，用来实现不同的写入方式。

handler的定义如下：

    type Handler interface {
        Write(p []byte) (n int, err error)
        Close() error
    }

我们可以参考python的log模块，实现几个通用的handler，譬如StreamHandler，FileHandler，TimeRotatingFileHandler。

log模块的实现[https://github.com/siddontang/golib/tree/master/log](https://github.com/siddontang/golib/tree/master/log)。
