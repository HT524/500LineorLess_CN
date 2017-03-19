[Greg Wilson](https://twitter.com/gvwilson)是 Software Carpentry, 一个科学家和工程师的计算技巧速成班，的创始人。他在工业界和学术界工作了三十余年，是好几本计算相关图书的作者或编辑，包括了 2008 年 Jolt 图书奖得主 *Beautiful Code* 和 *开源软件架构* 的前两卷。1993 年，Greg 获得了爱丁堡大学的计算机博士学位。



## 简介

在过去的二十多年里，网络改变了社会的各个方面，但它的核心却改动不多。大多数系统仍然遵循着 Tim Berners-Lee 在 25 年前所制定的规则。尤其是，大多数 Web 服务器仍旧以相同的方式处理着相同的数据，一如既往。

本章节将探讨他们如何实现。与此同时，本章节还将探讨开发者如何创建增加新特性而不需要重写的软件系统。


## 背景

几乎所有的网络程序都运行在一类叫做 互联网协议（IP）的通信标准上。这类协议中，我们涉及的是传输控制协议（TCP/IP），该协议使得计算机间通信类似于读写文件。


程序通过套接字使用 IP 协议进行通信。每个套接字是点对点通信信道的一端，正如电话机是一次电话通信的一端。一个套接字包含着一个 IP 地址，该地址确定了一台确定的机器和该机器上的一个端口号。IP 地址包含了四个八位数字，比如 `174.136.14.108`；域名系统将这些数字与字符相匹配，比如 `aosabook.org`，以便于记忆。


端口号码是 0 - 65535 之间的一个随机数，唯一确定了主机上的套接字。（如果说 IP 地址像一家公司的电话号码，那么端口号就像是分机号。）端口 0 - 1023  预留给操作系统使用；任何人都可以使用剩下的端口。

超文本传输协议（HTTP）描述了程序通过 IP 协议交换数据的一种方法。HTTP 协议刻意设计得简单: 客户端通过套接字发送一个请求，指定请求的东西，服务器在响应中返回一些数据（如下图）。该数据或许复制自硬盘上的文件，或许由程序动态生成，或是二者的混合。

![The HTTP Cycle](./http-cycle.png)

关于 HTTP 请求，最重要的地方在于，它仅由文本组成。任何有意愿的程序都可以对其进行创建或解析。不过，为了被正确地解析，文本中必须包含下图所展示的部分。

![An HTTP Request](./http-request.png)

(注: 'sp':空格, 'cr lf':换行)
HTTP 方法大多是 GET（请求信息）或者 POST（提交表单或上传文件）。统一资源定位器（URL）确定了客户端所请求的文件路径，一般位于硬盘上，比如 `/research/experiments.html`, 但是（接下来才是关键），如何处理完全取决于服务器。HTTP 版本一般是 "HTTP/1.0" 或 "HTTP/1.1" ; 二者之间的差异对我们来说并不重要。

HTTP 首部（Headers）是一组键值对，如同下面这三行：

```
Accept: text/html
Accept-Language: en, fr
If-Modified-Since: 16-May-2005
```

不同于哈希表中的键，HTTP 首部中，键可以出现任意多次。这将允许请求做一些事，例如指定愿意接收多种类型的内容。

最后，请求的主体是与请求关联的任何数据。这个应用于通过表单提交数据，上传文件等。首部的末尾和主体的开头之间必须由一个空行，以声明首部的结束。

首部中， `Content-Lenght` 告诉服务器在请求主体中有多少字节需要被读取。

HTTP 响应的格式与 HTTP 请求类似:

![An HTTP Response](./http-response.png)

版本号，首部，主体有着相同的格式和意义。状态码是一个数字，用来指示在处理请求时所发生的事情: 200 意味着 "一切工作正常"，404 意味着 "没有找到"，其他状态码也分别有着各自的含义。 状态词以易读的形式重复着上述信息，比如 "一切正常" 或是 "没有找到"。

本节中，我们只需要了解关于 HTTP 的两件事情。

第一，HTTP 是无状态的: 每个请求自行处理，服务器在两个请求之间不会记住任何东西。如果应用想要跟踪一些信息，比如用户的身份，它必须自己实现。

实现的方法通常使用 cookie, 这是服务器发送到客户端的短字符串，之后由客户端返回给服务器。当用户执行一些函数，需要在多个请求之间保存状态时，服务器会创建一个新的 cookie，将它存储在数据库中，然后发送给浏览器。每次浏览器返回 cookie，服务器通过 cookie 寻找关于用户行为的信息。

我们需要了解的第二点是，可以填充参数以提供更多的信息。比如说，如果我们使用搜索引擎，我们需要指定关键词。我们可以将这些附加到 URL 路径中，但更应该是在 URL 中附加参数。我们在 URL 后附加 '?' ，之后是以 '&' 分隔的键值对（'key=value'）。比如说，URL `http://www.google.ca?q=Python` 要求谷歌查询关于 Python 的页面: 键是字母 'q'，值是 'Python'。长一点的查询 `http://www.google.ca/search?q=Python&amp;client=Firefox`，告诉谷歌我们在使用 Firefox，诸如此类。我们可以传输任何参数，不过，哪些参数需要注意，如何解释这些参数，完全取决于网站上运行的程序。

当然，如果 '?' 和 '&' 用作特殊字符，必然有方法加以避免，正如必须有方法将一个双引号字符放置在由双引号分隔的字符串内。URL 编码标准使用 '%' 后跟两位代码表示特殊字符，并使用 '+' 字符替代空格。因此，我们使用 URL `http://www.google.ca/search?q=grade+%3D+A%2B` 在谷歌中搜索 "grade&nbsp;=&nbsp;A+"（注意空格）。

打开套接字，构建 HTTP 请求，解析响应极其乏味，因此大多数用户使用库来做大部分工作。Python 附带了一个这样的库，叫做 `urllib2`（因为它是之前的库 `urllib` 的代替者），但是它暴露了许多大多数用户不关心的东西。相比于 `urllib2`，`Requests` 库是一个更加易于使用的选择。接下来是一个例子，使用 Requests 下载来自 AOSA book 站点的一个页面。

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

`requests.get` 向服务器发送一个 HTTP GET 请求，返回一个包含响应的对象。该对象的 `status_code` 是响应的状态码；它的 `content_length` 是响应数据的字节数； `text` 是真正的数据（在这个例子中，是一个 HTML 页面）。

## Hello, Web

现在，我们已经为编写我们第一个简单的 Web 服务器做好了准备。基本思想很简单:

1. 等待用户连接我们的站点并发送一个 HTTP 请求；
2. 解析请求；
3. 计算出它所请求的；
4. 获取数据（或动态生成）；
5. 格式化数据为 HTML；
6. 返回数据。

步骤 1, 2, 6 都是从一个应用程序到另一个，Python 标准库有一个 'BaseHTTPServer' 模块，为我们实现这部分。我们只需要关心步骤 3 - 5，这也是我们在下面的小程序中所做的。

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

库里面的 `BaseHTTPRequestHandler` 类负责解析传进来的 HTTP 请求，并判断请求包含的方法。如果方法是 GET, 类将调用 `do_GET` 方法。我们的类 `RequestHandler` 重写了该方法以动态生成一个简单的页面: 文本页面存储在类级别变量中，我们将在发送给客户端 200 响应码，首部 `Content-Type` 字段以告诉客户端将返回的数据解析为 HTML，页面长度之后发送它。（`end_headers` 方法调用 插入空行以分隔首部和页面本身。）

然而 `RequestHandler` 并非故事的所有: 我们仍需要最后的三行来真正启动服务器。第一行以一个元组定义了服务器地址: 空字符串表示 "在当前主机上运行", 8080 标识了端口。接下来我们以地址和我们的请求处理类名作为参数创建了  `BaseHTTPServer.HTTPServer` 的一个实例，然后要求它一直运行（这意味着它将一直运行直至我们使用 'Ctrl - C' 杀掉它）。

如果我们在命令行中运行这个程序，它将不会显示任何东西：

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

第一行很简单: 因为我们没有要求一个特定的文件，浏览器便请求 '/'（任何正常工作服务器的根目录）。第二行出现是因为浏览器自动发送第二个请求，请求一个叫做 '/favicon.ico' 的图像文件，如果存在，将在地址栏显示为一个图标。


## 展示一些值

让我们修改我们的 Web 服务器以展示一些包含在 HTTP 请求中的值。（在调试时，我们会经常这样做，所以不妨先做一些练习。）为了保持代码整洁，我们将分别创建网页生成和发送。


```python
class RequestHandler(BaseHTTPServer.BaseHTTPRequestHandler):
    # ...page template...
    def do_GET(self):
        page = self.create_page()
        self.send_page(page)
    def create_page(self):
        # ...fill in...
    def send_page(self, page):
        # ...fill in...
```

`send_page` 比之前的多很多。

```python
    def send_page(self, page):
        self.send_response(200)
        self.send_header("Content-type", "text/html")
        self.send_header("Content-Length", str(len(page)))
        self.end_headers()
        self.wfile.write(page)
```

我们想要展示的页面的模板只是一个字符串，包含着一个有一些占位符的表格。

```python
    Page = '''\
<html>
<body>
<table>
<tr>  <td>Header</td>         <td>Value</td>          </tr>
<tr>  <td>Date and time</td>  <td>{date_time}</td>    </tr>
<tr>  <td>Client host</td>    <td>{client_host}</td>  </tr>
<tr>  <td>Client port</td>    <td>{client_port}s</td> </tr>
<tr>  <td>Command</td>        <td>{command}</td>      </tr>
<tr>  <td>Path</td>           <td>{path}</td>         </tr>
</table>
</body>
</html>
'''
```

填充表格的方法如下:

```python
    def create_page(self):
        values = {
            'date_time'   : self.date_time_string(),
            'client_host' : self.client_address[0],
            'client_port' : self.client_address[1],
            'command'     : self.command,
            'path'        : self.path
        }
        page = self.Page.format(**values)
        return page
```

该程序的主体并没有改变：正如之前，它以地址和请求处理程序作为参数，创建了一个 `HTTPServer` 类的实例，然后一直处理请求。如果我们运行它，然后用浏览器发送一个请求给 `http://localhost:8000/something.html` ,我们将得到:

```
  Date and time  Mon, 24 Feb 2014 17:17:12 GMT
  Client host    127.0.0.1
  Client port    54548
  Command        GET
  Path           /something.html
```

注意到，我们没有得到一个 404 错误，即使 `something.html` 页面并不存在。这是因为 Web 服务器只是一个程序，当它收到请求时，会做它所需要的任何事情: 返回之前请求提到的文件，提供一个随机选取的维基百科页面，或者我们编程时让它做的任何事情。

## 提供静态页面

显然，接下来的步骤是提供静态文件，取代动态生成。我们将重写 `do_GET`。

```python
    def do_GET(self):
        try:
            # Figure out what exactly is being requested.
            full_path = os.getcwd() + self.path
            # It doesn't exist...
            if not os.path.exists(full_path):
                raise ServerException("'{0}' not found".format(self.path))
            # ...it's a file...
            elif os.path.isfile(full_path):
                self.handle_file(full_path)
            # ...it's something we don't handle.
            else:
                raise ServerException("Unknown object '{0}'".format(self.path))
        # Handle errors.
        except Exception as msg:
            self.handle_error(msg)
```

上述方法假设允许程序使用所在路径（就是使用 `os.getcwd` 所得到的）下的任意文件提供服务。它会结合 URL 提供的路径（总是以 '/' 开始，`BaseHTTPServer` 会自动将它放入 `self.path`），以获取用户想要的文件的路径。如果文件不存在，或者路径并不指向文件，上述方法将通过获取并抛出异常来报告错误。另一方面，如果路径匹配到文件，`do_GET` 方法将调用辅助方法 `handle_file` 来读取并返回内容。辅助方法仅读取文件，然后调用 `send_content` 将文件内容返回给客户端：

```python
    def handle_file(self, full_path):
        try:
            with open(full_path, 'rb') as reader:
                content = reader.read()
            self.send_content(content)
        except IOError as msg:
            msg = "'{0}' cannot be read: {1}".format(self.path, msg)
            self.handle_error(msg)
```

请注意，我们以二进制方式打开文件—由 'rb' 中 'b' 标识，这样 Python 不会改变看起来像 Windows 行结束的字节序列。同时，请注意，在使用文件提供服务时，将整个文件读入内存在真实生活中并不合适，视频文件大小可能是好几G。处理上述情况已经超出了本章的范围。我们接下来编写错误处理方法和错误处理页面模板来结束本节。

```python
    Error_Page = """\
        <html>
        <body>
        <h1>Error accessing {path}</h1>
        <p>{msg}</p>
        </body>
        </html>
        """
    def handle_error(self, msg):
        content = self.Error_Page.format(path=self.path, msg=msg)
        self.send_content(content)
```

如果我们不仔细观察，程序似乎正常运行。问题在于它总是返回 200 状态码，即使所请求的页面并不存在。是的，返回的页面包含着错误信息，但因为浏览器读不懂英文，它并不知道请求实际失败了。为了使错误明确，我们需要修改 `handle_error` 和 `send_content` 如下:

```python
    # Handle unknown objects.
    def handle_error(self, msg):
        content = self.Error_Page.format(path=self.path, msg=msg)
        self.send_content(content, 404)
    # Send actual content.
    def send_content(self, content, status=200):
        self.send_response(status)
        self.send_header("Content-type", "text/html")
        self.send_header("Content-Length", str(len(content)))
        self.end_headers()
        self.wfile.write(content)
```

注意，当文件找不到时，我们并没有抛出 `ServerException` 异常，而是生成一个错误页面。`ServerException` 意味着服务器内部错误，即，我们弄错了。另一方面，当用户遇到错误时，此处即，请求了一个不存在的文件的 URL 时，由 `handle_error` 生成错误页面。

## 目录列表

下一步，我们将教会服务器，在 URL 中，路径代表目录而不是文件时，展示一个目录内容的列表。我们甚至可以更进一步，让程序在目录中寻找 `index.html` 文件来展示。不过在 `do_GET` 中构建这些方法或许是一个错误，因为生成的方法将会是很多 `if` 语句混杂在一起来控制特殊行为。正确的方案是退一步，解决一个更一般的问题：弄清楚如何处理一个 URL。

```python
    def do_GET(self):
        try:
            # Figure out what exactly is being requested.
            self.full_path = os.getcwd() + self.path
            # Figure out how to handle it.
            for case in self.Cases:
                handler = case()
                if handler.test(self):
                    handler.act(self)
                    break
        # Handle errors.
        except Exception as msg:
            self.handle_error(msg)
```

第一步完全相同：弄清楚请求的完整路径。之后，代码看起来就不同了。这个版本循环遍历一组存储在列表中的情况，而不是一组内嵌的测试。每种情况都是一个有着两个方法的对象，`test` 告诉我们是否能够处理请求，`act`，实际上进行处理。一旦我们发现正确的情况，我们让它处理请求并跳出循环。这三个类重复之前的服务器的行为：

```python
class case_no_file(object):
    '''File or directory does not exist.'''
    def test(self, handler):
        return not os.path.exists(handler.full_path)
    def act(self, handler):
        raise ServerException("'{0}' not found".format(handler.path))
class case_existing_file(object):
    '''File exists.'''
    def test(self, handler):
        return os.path.isfile(handler.full_path)
    def act(self, handler):
        handler.handle_file(handler.full_path)
class case_always_fail(object):
    '''Base case if nothing else worked.'''
    def test(self, handler):
        return True
    def act(self, handler):
        raise ServerException("Unknown object '{0}'".format(handler.path))
```

这是我们在 `RequestHandler` 类开头，如何构建事件处理程序列表：

```python
class RequestHandler(BaseHTTPServer.BaseHTTPRequestHandler):
    '''
    If the requested path maps to a file, that file is served.
    If anything goes wrong, an error page is constructed.
    '''
    Cases = [case_no_file(),
             case_existing_file(),
             case_always_fail()]
    ...everything else as before...
```

现在，表面上我们的服务器更加复杂了，而不是简洁。文件从 74 行变成 99 行，并有了一个额外的，没有任何新功能的间接层。不过当我们回到本节最初提出的任务:：教会服务器为一个目录请求，在 `index.html` 存在时返回 `index.html`， 不存在时返回目录内容列表，好处随之出现。前者的处理程序如下:

```python
class case_directory_index_file(object):
    '''Serve index.html page for a directory.'''
    def index_path(self, handler):
        return os.path.join(handler.full_path, 'index.html')
    def test(self, handler):
        return os.path.isdir(handler.full_path) and \
               os.path.isfile(self.index_path(handler))
    def act(self, handler):
        handler.handle_file(self.index_path(handler))
```

接下来，辅助方法 `index_path` 构造 `index.html` 文件的路径；将它放进事件处理程序以防止主类 `RequestHandler` 的杂乱。`test` 检查路径是否是一个包含 `index.html` 页面的目录，`act` 要求主请求处理程序返回这个页面。`RequestHandler` 所需的唯一变化是将一个 `case_directory_index_file` 类加入我们的 `Cases` 列表:

```python
    Cases = [case_no_file(),
             case_existing_file(),
             case_directory_index_file(),
             case_always_fail()]
```

要是目录不包含一个 `index.html` 页面呢？`test` 和之前的一个一致，可是，`act` 方法呢？它应该变成什么样？

```python
class case_directory_no_index_file(object):
    '''Serve listing for a directory without an index.html page.'''
    def index_path(self, handler):
        return os.path.join(handler.full_path, 'index.html')
    def test(self, handler):
        return os.path.isdir(handler.full_path) and \
               not os.path.isfile(self.index_path(handler))
    def act(self, handler):
        ???
```

似乎我们把自己逼进了墙角。逻辑上讲，`act` 方法应该创建并返回目录列表，但我们现存的代码不允许那样:：`RequestHandler.do_GET` 调用 `act` 方法，却不期望它返回一个值。让我们暂时为 `RequestHandler` 增加一个方法以生成目录列表，然后使用事件处理器的 `act` 方法调用它:

```python
class case_directory_no_index_file(object):
    '''Serve listing for a directory without an index.html page.'''
    # ...index_path and test as above...
    def act(self, handler):
        handler.list_dir(handler.full_path)
class RequestHandler(BaseHTTPServer.BaseHTTPRequestHandler):
    # ...all the other code...
    # How to display a directory listing.
    Listing_Page = '''\
        <html>
        <body>
        <ul>
        {0}
        </ul>
        </body>
        </html>
        '''
    def list_dir(self, full_path):
        try:
            entries = os.listdir(full_path)
            bullets = ['<li>{0}</li>'.format(e)
                for e in entries if not e.startswith('.')]
            page = self.Listing_Page.format('\n'.join(bullets))
            self.send_content(page)
        except OSError as msg:
            msg = "'{0}' cannot be listed: {1}".format(self.path, msg)
            self.handle_error(msg)
```

## CGI 协议

理所当然，大多数人不想为了添加新的功能而编辑 web 服务器的源代码。为了将他们从编辑源码拯救出来，服务器一般都支持一种叫做公共网关接口（CGI）的机制，它为 web 服务器提供了一个标准的方式来运行外部程序，以响应请求。例如，假设我们想要服务器可以在一个 HTML 页面上展示本地时间，我们可以在一个只有几行代码的独立程序中实现:

```python
from datetime import datetime
print '''\
<html>
<body>
<p>Generated {0}</p>
</body>
</html>'''.format(datetime.now())
```

为了让 web 服务器运行这个程序，我们添加了下面的事件处理器：

```python
class case_cgi_file(object):
    '''Something runnable.'''
    def test(self, handler):
        return os.path.isfile(handler.full_path) and \
               handler.full_path.endswith('.py')
    def act(self, handler):
        handler.run_cgi(handler.full_path)
```

`test` 很简单:：文件路径是否以 `.py` 结尾？是的话，`RequestHandler` 将运行上面的独立程序。

```python
    def run_cgi(self, full_path):
        cmd = "python " + full_path
        child_stdin, child_stdout = os.popen2(cmd)
        child_stdin.close()
        data = child_stdout.read()
        child_stdout.close()
        self.send_content(data)
```

这是非常不安全的: 如果有人知道了我们服务器上一个 Python 文件的路径，我们将不得不允许他运行该程序，而没有考虑，它有权限访问哪些数据，它是否包含一个死循环，或者二者之外。

扫清上述隐患，核心理念很简单:

1. 在一个子进程中运行该程序。
2. 捕获子进程发送到标准输出的一切。
3. 返回给发起请求的客户端。

完整的 CGI 协议比这更丰富，它允许 URL 中存在参数，服务器会将它们传入正在运行的程序，但这些细节并不会影响系统的整体架构...

事情再一次变得复杂。 `RequestHandler` 最初有一个方法，`handle_file`，来处理内容。我们现在在 `list_dir` 和 `run_cgi` 中增加两个特殊的情况。这三个方法无所谓在哪儿，因为它们主要由其他方法调用。

解决办法很简单: 为我们所有的事件处理器创建一个父类，当且仅当方法在多个处理器间共享时，将他们移入父类中。这样做了之后，`RequestHandler` 类就像下面这样:

``` python
class RequestHandler(BaseHTTPServer.BaseHTTPRequestHandler):
    Cases = [case_no_file(),
             case_cgi_file(),
             case_existing_file(),
             case_directory_index_file(),
             case_directory_no_index_file(),
             case_always_fail()]
    # How to display an error.
    Error_Page = """\
        <html>
        <body>
        <h1>Error accessing {path}</h1>
        <p>{msg}</p>
        </body>
        </html>
        """
    # Classify and handle request.
    def do_GET(self):
        try:
            # Figure out what exactly is being requested.
            self.full_path = os.getcwd() + self.path
            # Figure out how to handle it.
            for case in self.Cases:
                if case.test(self):
                    case.act(self)
                    break
        # Handle errors.
        except Exception as msg:
            self.handle_error(msg)
    # Handle unknown objects.
    def handle_error(self, msg):
        content = self.Error_Page.format(path=self.path, msg=msg)
        self.send_content(content, 404)
    # Send actual content.
    def send_content(self, content, status=200):
        self.send_response(status)
        self.send_header("Content-type", "text/html")
        self.send_header("Content-Length", str(len(content)))
        self.end_headers()
        self.wfile.write(content)
```

我们的事件处理程序的父类如下:

```python
class base_case(object):
    '''Parent for case handlers.'''
    def handle_file(self, handler, full_path):
        try:
            with open(full_path, 'rb') as reader:
                content = reader.read()
            handler.send_content(content)
        except IOError as msg:
            msg = "'{0}' cannot be read: {1}".format(full_path, msg)
            handler.handle_error(msg)
    def index_path(self, handler):
        return os.path.join(handler.full_path, 'index.html')
    def test(self, handler):
        assert False, 'Not implemented.'
    def act(self, handler):
        assert False, 'Not implemented.'
```

现存文件处理程序如下（随机选择一个例子）:

```python
class case_existing_file(base_case):
    '''File exists.'''
    def test(self, handler):
        return os.path.isfile(handler.full_path)
    def act(self, handler):
        self.handle_file(handler, handler.full_path)
```

## 讨论

原生的代码和重构后版本的差异反映了两个很重要的观念。第一个是把类看作是相关服务的一个集合。`RequestHandler` 和 `base_case` 并不作出决定或采取行动；它们只为其他做这些事的类提供工具。

第二个是可拓展性: 人们可以通过写一个外部的 CGI 程序，或者增加一个事件处理类，来为我们的 web 服务器增加新的功能。后者需要在 `RequestHandler` 中改变一行（将事件处理器插入事件列表），但我们可以让 web 服务器读一个配置文件，并从中加载事件处理类来摆脱上述改变。在这两种情况下，他们可以忽略大部分低层次细节，正如 `BaseHTTPRequestHandler` 类的开发者允许我们忽略处理套接字连接的细节和解析 HTTP 请求。

这些观念通常很有用；试试看你能否找到方法，将他们应用到你自己的项目中。

---

1. 我们将要在本章中用到好几次 `handle_error` ，包括一些 404 状态码不合适的情况。在你阅读的过程中，试着去思考，你将如何扩展这个项目，能使得状态码可以很轻松地在每种情况下提供。

2. 我们的代码也使用了 `popen2` 库函数，为了更好的支持子流程模块它被弃用。不过，`popen2` 用在这个例子中，的确是更少分散注意力的工具。
