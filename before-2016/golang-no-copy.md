在go里面，string和slice的互换是需要进行内存拷贝的，虽然在底层，它们都只是用 pointer + len来表示的一段内存。

通常，我们不会在意string和slice的转换带来的内存拷贝性能问题，但是总有些地方需要关注的，刚好在看vitess代码的时候，发现了一种很hack的做法，string和slice的转换只需要拷贝底层的指针，而不是内存拷贝。当然这样做的风险各位就要好好担当了：

    func String(b []byte) (s string) {
        pbytes := (*reflect.SliceHeader)(unsafe.Pointer(&b))
        pstring := (*reflect.StringHeader)(unsafe.Pointer(&s))
        pstring.Data = pbytes.Data
        pstring.Len = pbytes.Len
        return
    }
    
    func Slice(s string) (b []byte) {
        pbytes := (*reflect.SliceHeader)(unsafe.Pointer(&b))
        pstring := (*reflect.StringHeader)(unsafe.Pointer(&s))
        pbytes.Data = pstring.Data
        pbytes.Len = pstring.Len
        pbytes.Cap = pstring.Len
        return
    }
    
在我的测试例子中，slice转string之后，如果slice的值有变化，string也会跟着改变，如下：

    b := []byte("hello world")
    
    a := String(b)
    
    b[0] = 'a'
    
    println(a)  //output  aello world
    
但是string转slice之后，就不能更改slice了，如下：

    a := "hello world"
    
    b := Slice(a)
    
    b[0] = 'a'  //这里就等着崩溃吧
    
    //但是可以这样，因为go又重新给b分配了内存
    b = append(b, "hello world"…)
    
上面为什么会崩溃我猜想可能是string是immutable的，可能对应的内存地址也是不允许改动的。

另外，上面这个崩溃在defer里面是recover不回来的，真的就崩溃了，原因可能就跟c的非法内存访问一样，os不跟你玩了。