# 证书，CSP与Openssl

## 起因

最近在研究更安全的交互体系，自然想到的就是提供证书的交互方式。给用户分配一对公私钥，然后将私钥交给用户保管，用户在登录或者一些关键操作的时候通过私钥签名，从而保证其安全性。

鉴于团队的童鞋都没有开发usb key相关的经验，所以最开始的版本只考虑通过软证书实现。为了保证安全性，我们将用户的证书信息放置在windows系统的证书存储区里面，这样既减少证书被盗用的风险，同时通过windows的CSP（Cryptographic Service Provider）能让JS，APP都能读取到相关证书信息。

## 流程

当用户登录系统的时候，需要使用该用户对应的私钥进行签名，如果该用户现阶段还没有私钥，则会引导用户跳转到证书申请页面进行申请。申请成功之后，系统自动为用户安装证书，然后用户就能够使用该证书与服务器进行交互了。

### 证书生成

我们在服务器端通过openssl创建相关证书，主要有以下几个流程：

- 创建私钥
	
		openssl genrsa -out private.key 1024

	我们通过genrsa生成了一个1024位的私钥

- 生成CA

		openssl req -new -x509 -days 3650 -key private.key -out username.crt -subj "/C=CN/ST=GZ/L=ZH/O=KSS/OU=KSS/CN=username"

	这里需要特别注意的是证书的CN信息，我们这里使用username来表示，这样CSP就能通过username，使用subjectName来查找相关证书了。为了保证username的私密性，我们也可以对username进行hmac处理。

- 提取公钥

		openssl rsa -in private.key -out public.key -outform PEM -pubout

	提取对应公钥信息，后续服务器会通过该公钥进行验证与加密。

- 生成pfx

		openssl pkcs12 -export -inkey private.key -in username.crt -password pass: -out username.pfx

	将私钥以及证书信息打包进pfx文件，并将该文件分发给用户。因为我们是通过脚本进行证书生成的处理，所以这里需要设置**'-password pass:'**，用来保证脚本不会被hang up。

证书生成之后，我们会将证书的相关信息存放到数据库，同时将pfx文件分发给用户。

### 证书导入

因为用户证书的申请是由我们自己的页面进行控制，所以我们通过JS就能将该证书信息下载下来，同时调用CSP相关接口将证书注册进windows的CA Store里面。

在注册证书的时候，需要注意，有可能存在同名的subjectName证书（这种可能出现在用户更换证书的情况），需要首先将其删除。

### 证书使用

当用户登录的时候，会做如下处理：

- 通过username找到相应的私钥，进行签名sign
- 服务器收到登录请求之后，通过username找到对应的公钥，进行验证verify
- 服务器验证通过，则会使用公钥加密一个key返回给用户
- 用户通过私钥对key进行解密，解开之后，后续的交互通过该key进行。

对于其他关键性操作，也使用私钥进行sign，保证其安全性。对于服务器的验证，加密来说，我们使用pycrypto进行，谁叫我们使用的是python开发。

## 一些坑

为了实现上述功能，我们栽了很多坑，这里记录一下，也供后续参考。

- 字节序问题，windows csp对于sign使用的是小端序，而openssl等都使用的是大端序。所以我们在处理的时候需要进行字节序转换。
- openssl rsautl，dgst。这是最坑爹的了，openssl的rsautl貌似已经被废弃了，所以verify的时候不能使用rsautl，只能用dgst才能保证csp sign的数据服务器能verify。但是对于加解密来说rsautl竟然又可以。神奇！
- Base64，JS对于二进制流的处理比较蛋疼，所以我们的接口都是使用base64编码的，但python的base64会有一个**'\n'**，这个就得我们手动去掉了。
- CSPICOM，原以为JS能调用CSPICOM就很方便了，但是CSPICOM在win7已经不支持，所以只能我们自己封装一个ActiveX，来调用CSP对应函数。

## end

CSP的相关代码在[这里](https://gist.github.com/siddontang/5792247)。需要注意的是，证书的添加以及删除需要使用管理员权限。另外，不得不吐槽一下WIN32的API，笔者是在没有Visual AssixtX写的代码，太辛苦了！

版权声明：自由转载-非商用-非衍生-保持署名 [Creative Commons BY-NC-ND 3.0](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)