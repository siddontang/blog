在web编程中，安全性是我们都必须面临的一个问题，包括cookie伪造，xsrf攻击等。tornado作为一个web framework，在安全性方面也提供了很多功能，这里简单介绍一下。

# cookie

在web编程中，浏览器经常使用cookie来保存相关用户信息，用于与server交互，但是cookie有很多安全问题，譬如cookie伪造。cookie有很多方式被修改，javascript，flash，以及browser plugin等，所以首先需要保证cookie不被修改。

## secure cookie

tornado提供了secure cookie机制来保证cookie不被修改。tornado使用一个密钥用来给cookie进行签名，用来保证cookie只能被服务器修改。因为密钥只有tornado server知道，所以其它应用程序是没办法修改cookie的值。

tornado使用set_secure_cookie和get_secure_cookie来设置和读取browser的cookie。使用secure cookie，只需要在tornado启动的时候设置cookie_secret就行了，如下：

    import tornado.web 
    import tornado.httpserver 
    import tornado.ioloop 

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            count = self.get_secure_cookie('count')
            count = (int(count) + 1) if count else 1

            self.set_secure_cookie('count', str(count))

            self.write(str(count))

    settings = {
        'cookie_secret' : 'S6Bp2cVjSAGFXDZqyOh+hfn/fpBnaEzFh22IVmCsVJQ='
    }

    application = tornado.web.Application([
        (r"/", MainHandler),
    ], **settings)

    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(8080)
    tornado.ioloop.IOLoop.instance().start()

cookie_secret的生成如下：

    import base64
    import uuid
    base64.b64encode(uuid.uuid4().bytes + uuid.uuid4().bytes)

## httponly 

为了防止cross-site scripting attack，tornado可以再设置cookie的时候增加httponly字段，这样该cookie就不能够被javascript读取。同时，为了更高的安全性，可以给cookie设置secure属性，这个跟先前讨论的set_secure_cookie不一样，上面那个是对cookie进行加密签名，保证不被修改，而这个则是让browser通过ssl传输cookie。启用这些功能很简单，只需要在设置cookie的时候处理。

    class MainHandler(tornado.web.RequestHandler):
        def get(self):
            self.set_cookie('count1', '1', httponly = True, secure = True)
            self.set_secure_cookie('count2', '2', httponly = True, secure = True)

# XSRF

上面介绍的方法虽然能够保证cookie的安全，但是还是防止不了[XSRF攻击](http://en.wikipedia.org/wiki/Cross-site_request_forgery)。为了防止XSRF攻击，首先设计web服务的时候就需要考虑http method的side effects。按照restful的编程模型，get只能用来获取数据，post才会去修改数据，这样就能在很大的程度上面防止XSRF，因为对于通常情况来说，XSRF攻击就是通过设置img的src为一个恶意的get请求，只要我们的get不会进行数据修改，自然就能防止。

但是一些恶意的操作仍然能够发送post请求来进行攻击，譬如通过HTML forms或者XMLHTTPRequest API，所以为了防止post这种的攻击，我们需要一些额外的机制。

tornado提供了一个XSRF保护机制，原理很简单，就是在post提交数据的时候额外加入一个_xsrf字段，这个字段的值是从secure cookie里面获取的，因为其它的应用程序获取不到这个cookie的值，所以我们能够保证post的安全性。开启xsrf protection也很简单。

    settings = {
        'cookie_secret' : 'S6Bp2cVjSAGFXDZqyOh+hfn/fpBnaEzFh22IVmCsVJQ=',
        'xsrf_cookies' : True
    }

    application = tornado.web.Application([
        (r"/", MainHandler),
        (r"/purchase", PurchaseHandler)
    ], **settings)


在post提交的form里面，我们只需要设置如下：

    <form action="/purchase" method="POST">
        {{ xsrf_form_html() }}
        <input type="submit" value="同意" name="agree"/>
    </form>


# User Authentication

tornado还提供了用户认证功能，当用户登录之后，会将用户的相关信息保存到一个地方，通常是在cookie里面。当用户的cookie过期，再次访问web服务的时候就会被重定向到登陆页面。

要实现上述功能，只需要重载get_current_user函数，配置login_url和使用authenticated decorator就行了。如下：

    class BaseHandler(tornado.web.RequestHandler):
        def get_current_user(self):
            return self.get_secure_cookie('username')

    class LoginHandler(BaseHandler):
        def get(self):
            str = '''<body><form action="/login" method="POST">
                    UserName: <input type="text" name="username" />
                              <input type="submit" value="Login" />
                    </form></body>'''
            self.write(str)

        def post(self):
            self.set_secure_cookie('username', self.get_argument('username'))
            self.redirect('/')

    class MainHandler(BaseHandler):
        @tornado.web.authenticated
        def get(self):
            user = self.current_user
            self.write('hello ' + user)

    class LogoutHandler(BaseHandler):
        def get(self):
            self.clear_cookie('username')
            self.redirect('/')

    settings = {
        'cookie_secret' : 'S6Bp2cVjSAGFXDZqyOh+hfn/fpBnaEzFh22IVmCsVJQ=',
        'login_url' : '/login'
    }

    application = tornado.web.Application([
        (r'/', MainHandler),
        (r'/login', LoginHandler),
        (r'/logout', LogoutHandler)
    ], **settings)

    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(8080)
    tornado.ioloop.IOLoop.instance().start()


# 总结

web安全一直是一个非常严重的问题，我们在写代码的时候一定要注意，这里有一篇如何写安全web的[Guide](https://wiki.mozilla.org/WebAppSec/Secure_Coding_Guidelines)。tornado虽然提供了一些安全机制，但是仍然不能完全保证绝对安全性，所以很多时候就需要我们决定我所写的服务到底应该有什么样的安全级别，从在这个级别下面如何保证安全性就可以了。