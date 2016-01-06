因为项目要支持国际化，最近跟一个同事在讨论多语言版本下面url如何设计，假如我们需要支持en和cn的版本。

他倾向于支持如下的url格式，后续以格式1指代：

    /en/group/abc.html
    /cn/group/abc.html
    
而我则倾向于只提供一套url，lang的信息在其它地方携带，后续以格式2指代，譬如：

    /group/abc.html?lang=en
    
    
对于格式1，它的好处在于非常清晰的就告知用户当前网页是什么语言的，但是我觉得还有几个不足：

- nginx location的适配，为了适配url里面的lang，我们需要将所有的location改成如下形式：

        /(.+)/group/abc.html

    改动比较大，同时nginx所有的location匹配都只能走正则匹配，性能稍微有点损耗。
 
- url跳转，假如用户现在在 /en/group/abc.html下面，切换成cn页面，如何正确的跳转到/cn/group/abc.html，也是需要考量的，我并不认为这个很容易，尤其是在nginx以及后端django都需要按照统一规则处理情况下。

虽然有很多网站都是采用格式1的方式来支持国际化，譬如apple，nginx的官网，但是我发现当它们在涉及语言切换的时候，很多都跳转到了该语言的首页，可能这样实现更简单吧（这绝对是我的臆想！！！）。

对于我倾向的格式2，想到的有三种方式带上lang的信息：

- 参数携带，譬如上面的/group/abc.html?lang=en
- cookie携带，譬如在cookie里面设置 lang=en
- Accept-Language，当没有任何lang信息的时候根据Accept-Language判断

当用户切换语言，或者url参数里面有lang信息的时候，我们会将该lang信息设置到cookie里面，这样下次访问的时候就可以根据cookie里面的访问了。

可以看到，格式2有明显的几个好处：

- url不变，无论增加多种语言，url仍然是同一个url，语言切换的时候也不会进行url跳转。
- restful，我认为，在restful模式中，lang并不是资源，而是资源用来展示的一种方式，这个跟Content-Type类似。

但是，格式2也有一些不足的地方，主要就在静态文件缓存，因为是同一个url，浏览器会对静态文件缓存，如果切换了lang，那么很有可能缓存的仍然是切换之前lang对应的文件。

总之，两种格式的url在业界都有使用，据我观察，google，facebook，twitter在处理多语言的时候，采用的是格式2的方式，而我们在经过讨论之后，也决定采用格式2的方式。

    
    