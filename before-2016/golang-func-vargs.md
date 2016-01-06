几天前纠结了一个蛋疼的问题，在go里面函数式支持可变参数的，譬如...T，go会创建一个slice，用来存放传入的可变参数，那么，如果创建一个slice，例如a，然后以a...这种方式传入，go会不会还会新建一个slice，将a的数据全部拷贝一份过去？

如果a很大，那么将会造成很严重的性能问题，不过后来想想，可能是自己多虑了，于是查看go的文档，发现如下东西：


> Passing arguments to ... parameters
> 
> If f is variadic with a final parameter p of type ...T, then within f the type of p is equivalent to type []T. If f is invoked with no actual arguments for p, the value passed to p is nil. Otherwise, the value passed is a new slice of type []T with a new underlying array whose successive elements are the actual arguments, which all must be assignable to T. The length and capacity of the slice is therefore the number of arguments bound to p and may differ for each call site.
> 
> Given the function and calls
> 
>     func Greeting(prefix string, who ...string)
>     Greeting("nobody")
>     Greeting("hello:", "Joe", "Anna", "Eileen")
> 
> within Greeting, who will have the value nil in the first call, and []string{"Joe", "Anna", "Eileen"} in the second.
> 
> If the final argument is assignable to a slice type []T, it may be passed unchanged as the value for a ...T parameter if the argument is followed by .... In this case no new slice is created.
> 
> Given the slice s and call
> 
>     s := []string{"James", "Jasmine"}
>     Greeting("goodbye:", s...)
> 
> within Greeting, who will have the same value as s with the same underlying array.

也就是说，如果我们传入的是slice...这种形式的参数，go不会创建新的slice。写了一个简单的例子验证：

    package main
    
    import "fmt"
    
    func t(args ...int) {
        fmt.Printf("%p\n", args)
    }
    
    func main() {
        a := []int{1,2,3}
        b := a[1:]
        
        t(a...)
        t(b...)
        
        fmt.Printf("%p\n", a)
        fmt.Printf("%p\n", b)
    }
    
    //output
    0x1052e120
    0x1052e124
    0x1052e120
    0x1052e124
    
可以看到，可变参数args的地址跟实际外部slice的地址一样，用的同一个slice。
