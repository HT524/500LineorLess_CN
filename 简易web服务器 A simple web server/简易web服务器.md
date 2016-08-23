> 原文: [A Simple Web Server](http://aosabook.org/en/500L/pages/a-simple-web-server.html)
> 作者: [Greg Wilson](https://twitter.com/gvwilson)


由于译者才疏学浅，翻译不当之处在所难免，望大家不吝指教。另外，本篇文章是我们开源项目 [500LineorLess_CN](https://github.com/HT524/500LineorLess_CN) 的一部分，欢迎大家共同参与！

---

[Greg Wilson](https://twitter.com/gvwilson)是 Software Carpentry, 一个科学家和工程师的计算技巧速成班，的创始人。他在工业界和学术界工作了三十年，是好几本关于计算的图书的作者或者编辑，包括了 2008 年 Jolt 图书奖得主 *Beautiful Code* 和 *开源软件架构* 的前两卷。Greg 在 1993 年获得了爱丁堡大学的计算机博士学位。



## 简介

在过去的二十多年里，网络改变了社会的各个方面，但它的核心却改动不多。大多数系统仍然遵循着 Tim Berners-Lee 在25年前所制定的规则。尤其是，大多数 Web 服务器仍旧以相同的方式处理着相同的数据，一如既往。

本章节将探讨他们如何做。与此同时，本章节还将探讨开发者如何创建不需要重写来增加新特性的软件系统。


## 背景

几乎网络上的所有程序都运行在一类叫做 互联网协议（IP）的通信标准上。这类协议中，我们涉及到的是传输控制协议（TCP/IP），该协议使得计算机间通信类似读写文件。


程序通过套接字使用 IP 协议进行通信。每个套接字是点对点通信信道的一端，正如电话机是一个电话通信的一端。一个套接字包含着一个 IP 地址，该地址确定了一台确定的机器和该机器上帝一个端口号。IP 地址包含了四个八位数字，比如说 `174.136.14.108`；域名系统将这些数字与字符名相匹配，比如 `aosabook.org`，以便于记忆。


端口号码是 0-65535之间的一个随机数，唯一确定了主机上的套接字。（如果说 IP 地址像一家公司的电话号码，那么端口号就像是分机号。）端口 0-1023 预留给操作系统使用；任何人都可以使用剩下的端口。

超文本传输协议（HTTP）描述了程序通过 IP 协议交换数据的一种方法。HTTP 协议刻意设计得简单: 客户端通过套接字发送一个请求，指定想要的东西，服务器在响应中返回一些数据（如下图）。该数据或许复制自硬盘上的文件，或许由程序动态生成，或者是二者的混合。

![The HTTP Cycle](http://ww3.sinaimg.cn/large/005PneI2gw1f71ac7tmsrj30gl05gdhc.jpg)

关于 HTTP 请求，最重要的地方在于，它仅由文本组成。任何有意愿的程序都可以对其进行创建或者解析。不过，为了被正确解析，文本中必须包含下图展示的部分。

![An HTTP Request](http://ww4.sinaimg.cn/large/005PneI2gw1f71aedvqu5j30c404naam.jpg)

HTTP 方法大多是 GET（请求信息）或者 POST（提交表单数据或者上传文件）。统一资源定位器（URL）确定了客户端所请求的，一般是硬盘上文件的路径，比如 `/research/experiments.html`, 但是（接下来才是关键），如何处理完全取决于服务器。HTTP 版本一般是 "HTTP/1.0" 或 "HTTP/1.1"; 二者之间的差异对我们来说并不重要。

HTTP 首部（Headers） 是一组键值对，如同下面这三行:

```
Accept: text/html
Accept-Language: en, fr
If-Modified-Since: 16-May-2005
```

不同于哈希表中的键，HTTP 首部中键可以出现任意多次。这将允许请求做一些事情，例如指定愿意接收多种类型的内容。

最后，请求的主体是与请求关联的任何数据。这应用于通过表单提交数据，上传文件等。首部的末尾和主体的开头之间必须由一个空行，以声明首部的结束。

首部中， `Content-Lenght` 告诉服务器在请求主体中有多少字节需要被读取。

HTTP 响应格式与 HTTP 请求类似:

![An HTTP Response](http://ww1.sinaimg.cn/large/005PneI2gw1f71afn3dm7j30dd0653zg.jpg)

版本号，首部，主体有着相同的格式和意义。状态码是一个数字，用来指示在处理请求时所发生的事情: 200 意味着 "一切工作正常"，404 意味着 "没有找到"，其他状态码也有着各自的含义。 状态词以易读的形式重复着上述信息，比如 "OK" 或 "没找到"。

本节中，我们只需要了解关于 HTTP 的两件事情。

第一个是，HTTP 是无状态的: 每个请求自行处理，服务器在两个请求之间不会记住任何事情。如果应用想要跟踪一些东西，比如用户的身份，它必须自己实现。

实现的方法通常的使用 cookie, 这是服务器发送到客户端的短字符串，之后客户端返回给服务器。当用户执行一些函数，需要在多个请求之间保存状态，服务器会创建一个新的 cookie，将它存储在数据库中，然后发送给浏览器。每次浏览器返回 cookie，服务器通过 cookie 寻找用户行为的信息。

我们需要了解的关于 HTTP 的第二件事情是，可以填充参数以提供更多的信息。比如说，如果我们使用搜索引擎，我们需要指定我们的关键词。我们可以将这些附加到 URL 路径中，但我们应该在 URL 中附加参数。我们在 URL 后附加 '?' ，之后是以 '&' 分隔的键值对（'key=value'）。比如说，URL `http://www.google.ca?q=Python` 要求谷歌查询关于 Python 的页面: 键是字母 'q'，值是 'Python'。长一点的查询 `http://www.google.ca/search?q=Python&amp;client=Firefox`，告诉谷歌我们在使用 Firefox(火狐浏览器), 诸如此类。我们可以传输任何参数，不过，哪些参数需要注意，如何解释这些参数，完全取决与网站上运行的程序。

当然，如果 '?' 和 '&' 用作特殊字符，必然由方法加以避免，正如必须有方法将一个双引号字符放置在由双引号分隔的字符串内。URL 编码标准使用 '%' 后跟两位代码表示特殊字符，并使用 '+' 字符替代空格。因此，我们使用 URL `http://www.google.ca/search?q=grade+%3D+A%2B` 在谷歌中搜索 "grade&nbsp;=&nbsp;A+"（注意空格）。

打开套接字，构建 HTTP 请求，解析响应极其乏味，因此大多数用户使用库来做大部分工作。Python 附带了一个这样的库，叫做 `urllib2`（因为它是之前的库 `urllib` 的代替者），但是它暴露了许多大多数用户不关心的管道。相比 `urllib2`，Request 库是一个更加易于使用的选择。接下来是一个例子，使用 Request 下载来自 AOSA book 站点的一个页面。

```python
import requests
response = requests.get('http://aosabook.org/en/500L/web-server/testpage.html')
print 'status code:', response.status_code
print 'content length:', response.headers['content-length']
print response.text
```

```
status code: 200
content length: 61
<html>
  <body>
    <p>Test page.</p>
  </body>
</html>
```

`request.get` 向服务器发送一个 HTTP GET 请求，返回一个包含响应的对象。该对象的 `status_code` 是响应的状态码；它的 `content_length` 是响应数据的字节数； `text` 是真正的数据（在这个例子中，是一个 HTML 页面）。

## Hello, Web

现在，我们已经为编写我们第一个简单的 Web 服务器做好了准备。基本思想很简单:

1. 等待用户连接我们的站点并发送一个 HTTP 请求;
2. 解析请求;
3. 计算出它所请求的;
4. 获取数据（或动态生成）;
5. 格式化数据为 HTML
6. 返回数据

步骤 1, 2, 6 都是从一个应用到另一个，Python 标准库有一个 'BaseHTTPServer' 模块，为我们实现这部分。我们只需要关心步骤 3-6，这也是我们在下面的小程序里面所做的。

```
import BaseHTTPServer
class RequestHandler(BaseHTTPServer.BaseHTTPRequestHandler):
    '''Handle HTTP requests by returning a fixed 'page'.'''
    # Page to send back.
    Page = '''\
<html>
<body>
<p>Hello, web!</p>
</body>
</html>
'''
    # Handle a GET request.
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/html")
        self.send_header("Content-Length", str(len(self.Page)))
        self.end_headers()
        self.wfile.write(self.Page)
#----------------------------------------------------------------------
if __name__ == '__main__':
    serverAddress = ('', 8080)
    server = BaseHTTPServer.HTTPServer(serverAddress, RequestHandler)
    server.serve_forever()
```

库里面的 `BaseHTTPRequestHandler` 类负责解析进来的 HTTP 请求，并判断请求包含的方法。如果方法是 GET, 类将调用 `do_GET` 方法。我们的类 RequestHandler 重写了该方法以动态生成一个简单的页面: 文本页面存储在类级别变量中，我们将在发送给客户端 200 响应码，首部 `Content-Type` 字段以告诉客户端将返回的数据解析为 HTML，页面长度之后发送它。（`end_headers` 方法调用 插入空行以分隔首部和页面本身。）

然而 RequestHandler 并非故事的所有: 我们仍需要最后的三行来真正启动服务器。第一行以一个元组定义了服务器地址: 空字符串表示 "在当前主机上运行", 8080 标识了端口。接下来我们以地址和我们的请求处理类名作为参数创建了  `BaseHTTPServer.HTTPServer` 的一个实例，然后要求它一直运行（这意味着它将一直运行直至我们使用 'Control-C' 杀掉它）。

如果我们在命令行中运行这个程序，它不会显示任何东西:

```
$ python server.py
```

如果我们在浏览器中访问 `http://localhost:8080`, 我们将在浏览器中看到:

```
Hello, web!
```

同时在 shell 中:

```
127.0.0.1 - - [24/Feb/2014 10:26:28] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [24/Feb/2014 10:26:28] "GET /favicon.ico HTTP/1.1" 200 -
```

第一行很简单: 因为我们没有要求一个特定的文件，浏览器便请求 '/'（任何正常工作服务器的根目录）。第二行出现是因为浏览器自动发送第二个请求，请求一个叫做 '/favicon.ico' 的图像文件，如果存在，它将在地址栏显示一个图标。
