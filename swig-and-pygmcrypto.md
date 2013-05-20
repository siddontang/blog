# swig的学习以及国密的python封装

## 起因

最近在研究国密算法，而我们主要是使用python来进行开发，所以就需要构建一个国密的python模块。

国密算法网上已经有很好的实现，笔者使用的是一个参考Xyssl实现的那个版本。因为这些版本都是c的，所以很容易将其扩展到python里面，但是为了跟python自身的crypto的行为一致，需要将国密生成相应的class。譬如，python的hashlib的md5，使用方式如下：

    md = hashlib.md5("123")
    md.update("456")
    print md.hexdigest()
    mdCopy = md.copy()
    print mdCopy.digest()
    
所以，对于类似的国密sm3算法，我们扩展到python里面，也应该提供如上的使用方式。

注册一个class到python，也不是一件很困难的事情，但是笔者觉得可以采用更简单的方式，自然就考虑使用swig。

## swig

[swig](http://www.swig.org/)（需翻墙）可以算是将c、c++代码生成到其他语言的一个代码生成器，

网上关于swig的学习介绍有很多，这里笔者只是记录一下写python的国密扩展时候使用到的swig知识点。

以sm3来说，为了以class的方式在python里面使用，我们首先需要在swig定义其class的template，如下：
    
    class SM3 
    {
    public:
        SM3(const unsigned char* input, int inputLen);
        ~SM3();
    
        void update(const unsigned char* input, int inputLen);
        void digest(unsigned char output[32]);
        void hexdigest(unsigned char output[64]);
        SM3 copy();
    };
   
在python里面使用的时候，我们可以使用如下方式创建一个sm3对象：
    
    md = SM3("1234567890")
   
可以看到，在python里面，我们创建sm3对象只传入了一个参数，那么怎么对应c++里面的这两个(unsigned char*, int)呢？这里，我们使用swig里面的[typemaps](http://www.swig.org/Doc2.0/Typemaps.html#Typemaps)：

    %typemap(in) (const unsigned char* input, int inputLen) 
    {
        $1 = (unsigned char*)PyString_AsString($input);
        $2 = PyString_Size($input);
    }
   
这里，我们定义一个in typemap，swig碰到这种typemap会将python里的函数参数转换成对应的c++参数。

对于sm3的digest以及hexdigest，python里面如下：

    print md.digest()
    print md.hexdigest()
    
可以看到，在c++里面，output是函数的参数，而在python里面则是作为返回值。这里，我们可以使用argout typemap来处理。

    %typemap(in, numinputs = 0) unsigned char [ANY] (unsigned char temp[$1_dim0]) 
    {
        $1 = temp;
    }
    
    %typemap(argout) unsigned char [ANY] 
    {
        Py_XDECREF($result);
        $result = PyString_FromStringAndSize((const char*)$1, $1_dim0);
    }
    
这里，我们需要注意几个地方:

- 因为digest以及hexdigest的array长度是32和64，所以使用ANY来匹配，使用$1_dim0来获取实际的array长度。
- 因为digest以及hexdigest在python里面是没有参数的，所以我们需要swig忽略输入参数，使用numinputs = 0。
- 因为没有参数传入，所以我们需要手动构造一个array，使用unsigned char temp[$1_dim0]，然后将这个array的地址赋给实际的参数output。
- 使用argout，将output的结果放到python函数的返回值。

上述的实现，是参考[multi_argument_typemaps](http://www.swig.org/Doc2.0/Typemaps.html#Typemaps_multi_argument_typemaps)里面的很多例子，尤其是关于in buffer和out buffer的。

## pygmcrypto

对于国密算法，笔者这里只使用了sm3和sm4，对于sm2来说，笔者认为太过复杂，还没有较好的开源实现，同时笔者也没有精力自己去写一个，如果有谁有好的实现，欢迎联系我，笔者乐意将其build进python模块里去。

代码[pygmcrypto](https://github.com/siddontang/pygmcrypto)在这里。直接使用python setup.py install进行安装，使用如下:

    from gmcrypto import sm4, sm3
    md = sm4.new("1" * 16)
    eData = md.encrypt("1" * 16)
    dData = md.decrypt(dData)
    
    md = sm3.new("123")
    md.update("456")
    md.hexdigest()

如果出现编译不成功的情况，很有可能是生成的swig template问题，大家只需要在swig目录下面make重新生成就可以了。swig的版本需要2.0以上。

版权声明：自由转载-非商用-非衍生-保持署名 [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)
