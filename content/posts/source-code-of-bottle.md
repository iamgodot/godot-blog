---
title: "Bottle 框架源码阅读"
date: 2022-04-22T21:53:29+08:00
draft: true
mermaid: true
categories:
  - Code
tags:
  - python
keywords:
  - Bottle 源码阅读
  - Python bottle 框架
  - Python bottle framework
---

写这篇文章最开心的一点是终于可以用这张截图了：

![](https://static.iamgodot.com/content/images/20220423155637.png)

相比名声在外的 Django/Flask/FastAPI，Bottle 可以说是非常不起眼了，甚至很多人并不知道它的存在。其实在很多方面，这个框架都极其优秀：

- 速度：截止到 2022-04-13，Bottle 在一众 Python Web 框架的[测评](https://web-frameworks-benchmark.netlify.app/result?l=python)中名列第二，要知道这可是十年以上的老前辈了。
- 易用性：Bottle 早在 Flask 之前就使用了装饰器来定义路由，此外还有全局可用的 Request/Response 对象。
- 文档：不仅将框架本身的使用讲得很清楚，还总结了很多 Web 场景下的解决方案。
- 代码质量：虽然为了 Python 2 做了不少兼容，但是代码很精炼，而且 Pythonic。
- 其他：Bottle 坚持单模块以及无第三方库依赖；仓库仍然在积极维护中。

换作几年前，我会一开始就使用并将 Bottle 研究透彻，而不是让自己淹没在 Django 浩瀚如烟的文档中。下面开始梳理 Bottle 源码的阅读理解。因为代码量不大，所以就直接看最新的版本了：`0.11.1 - 5a6c620`。

# Web 框架的基本元素

参考 [The Hitchhiker's Guide to Python](https://docs.python-guide.org/scenarios/web/) 的说法，一个 Web 框架要满足的基本功能：

1. URL Routing
2. Request and Response Objects
3. ~~Template Engine~~
4. Development Web Server

从后端的角度来讲更重要的是 1、2、4 三项，其中 1 负责转发请求到对应的视图函数，2 是对 HTTP 协议元素的解析处理，而 4 决定了服务的部署方式和基础性能。

Bottle 在这几方面都做了很好的实现：路由上提供了通配符匹配和装饰器接口；请求和响应对象作为全局对象存在并保证了线程安全；Server 部署除了 Python 自带的 `wsgiref` 还支持[绝大多数的 WSGI Server](https://bottlepy.org/docs/dev/deployment.html#switching-the-server-backend)。

之外 Bottle 服务还会自动检测代码变更并重启，扩展方面有 Hook 和 Plugin 机制等等。

# 一切从 WSGI 开始

WSGI 定义了 Python Web 框架的统一接口规范，因此也是了解 Bottle 最好的突破口。一个 Web 服务可以简单看成 Server 和 Handler 两部分，前者负责监听端口并建立连接，而后者处理请求然后返回响应内容。Handler 在 WSGI 中称为 app，是一个可调用对象，比如：

```python
def application(environ, start_response):
    response_body = [
        '%s: %s' % (key, value) for key, value in sorted(environ.items())
    ]
    response_body = '\n'.join(response_body)

    status = '200 OK'
    response_headers = [
        ('Content-Type', 'text/plain'),
        ('Content-Length', str(len(response_body)))
    ]
    start_response(status, response_headers)

    return [response_body.encode()]
```

在 Bottle 中 app 被包装成了一个 `Bottle` 对象，实际的调用过程在它的 `wsgi` 方法里：

```python
class Bottle(object):
    ...
    def wsgi(self, environ, start_response):
        try:
            # 获取响应内容并做适当转化
            out = self._cast(self._handle(environ))
            ...
            exc_info = environ.get("bottle.exc_info")
            if exc_info is not None:
                del environ["bottle.exc_info"]
            start_response(response._wsgi_status_line(), response.headerlist, exc_info)
            return out
        except (KeyboardInterrupt, SystemExit, MemoryError):
            ...

    def _handle(self, environ):
        ...
        try:
            while True:
                out = None
                try:
                    self.trigger_hook("before_request")
                    # 通过路由找到视图函数
                    route, args = self.router.match(environ)
                    environ["route.handle"] = route
                    environ["bottle.route"] = route
                    environ["route.url_args"] = args
                    # 调用视图函数获取结果
                    out = route.call(**args)
                    break
                except HTTPResponse as E:
                    ....
```

可以看到，`_handle` 方法负责路由到对应的视图函数并调用获取响应，而 `_cast` 会对响应内容进行 WSGI 兼容的处理。

接下来根据 `self.router.match(environ)` 来看路由部分的具体实现。

# 路由

`Router` 是抽象出来的负责路由转发的对象，实质上是一系列 Routes 的集合，而每个 `Route` 代表方法和路径到视图函数的对应关系。因此，当一个 HTTP 请求到来，Router 就可以从 Routes 中找到匹配的视图函数。思路很简单，关键是如何实现高效的查找过程。下面看 `Router` 的代码：

```python
class Router(object):
    ...
    def match(self, environ):
        ...
        for method in methods:
            if method in self.static and path in self.static[method]:
                target, getargs = self.static[method][path]
                return target, getargs(path) if getargs else {}
            elif method in self.dyna_regexes:
                for combined, rules in self.dyna_regexes[method]:
                    match = combined(path)
                    if match:
                        target, getargs = rules[match.lastindex - 1]
                        return target, getargs(path) if getargs else {}
        ...
```

核心逻辑有两部分：首先是静态匹配，在 `self.static` 中保存了从方法到路径再到视图函数（`target`）的映射，这里会直接用哈希表实现快速查找；其次是通配符匹配，`self.dyna_regexes` 里每个方法都包含多个 `(combined, rules)` 元组，而 `combined` 由多个正则表达式组合到一起（每个正则代表一个动态路径），对应到 `rules` 中的多个视图函数。之所以会有多个元组，是因为 CPython 中正则的分组匹配最多只支持 99 个，所以一个 `combined` 的容量是有限的，如果动态路由过多，就需要增加新的元组。

这里体现了 Bottle 路由查找的基本原则：

- 静态路由匹配的优先级高于动态路由。
- 动态路由匹配有先后顺序，注意不要造成短路。
- 还有添加路由的时候
  - 一个视图函数可以定义多个路径。
  - 重复定义会覆盖原有路由，当然这也是允许的。

# 请求 & 响应

接下来是对 HTTP 请求和响应的抽象，Bottle 定义了全局的 Request & Response 对象。看起来和 Flask 很像，其实要比后者更早。

关键在于保证线程安全，这部分已经在 [Python 中的 TLS 是如何实现的](/posts/sourcecode-of-python-threadlocal) 中说得很详细了，下面看看 Bottle 是怎么实现的：

```python
def _local_property():
    ls = threading.local()

    def fget(_):
        try:
            return ls.var
        except AttributeError:
            raise RuntimeError("Request context not initialized.")

    def fset(_, value):
        ls.var = value

    def fdel(_):
        del ls.var

    return property(fget, fset, fdel, "Thread-local property")

class LocalRequest(BaseRequest):
    bind = BaseRequest.__init__
    environ = _local_property()

class LocalResponse(BaseResponse):
    bind = BaseResponse.__init__
    _status_line = _local_property()
    _status_code = _local_property()
    _cookies = _local_property()
    _headers = _local_property()
    body = _local_property()

...

request = LocalRequest()
response = LocalResponse()
```

基于 `threading.local`，Bottle 使用修饰符来定义 `LocalRequest` 和 `LocalResponse` 中的 HTTP 属性，这样的实现很灵活，也可以轻松地应用到其他的对象。

需要注意的是，在 greenlet 和 coroutine 大行其道的今天，`threading.local` 已经完全不够用了。Bottle 与 ASGI 水土不服，但是一定要部署成 Async 服务的话，也是有方法的：比如保证 `threading.local` 提前被 monkeypatch（针对 gevent），或者在代码中使用 `request.copy()` 拷贝出新的请求对象。作者在文档和 [Issue](https://github.com/bottlepy/bottle/issues?q=async) 中都做了详细的解释。

# 服务

Bottle 默认使用了 `wsgiref` 模块中的 WSGI Server 来启动服务，这种服务是单线程的，所以也只适用于本地开发。

此外 Bottle 支持许多 Web Server 部署，在官网有详细列举：

![](https://static.iamgodot.com/content/images/20220423123506.png)

为了兼容这么多的 Server，Bottle 在内部实现了各种各样的适配器，比如 `wsgiref`：

```python
class ServerAdapter(object):
    quiet = False

    def __init__(self, host="127.0.0.1", port=8080, **options):
        self.options = options
        self.host = host
        self.port = int(port)

    def run(self, handler):
        pass

    def __repr__(self):
        args = ", ".join("%s=%s" % (k, repr(v)) for k, v in self.options.items())
        return "%s(%s)" % (self.__class__.__name__, args)

class WSGIRefServer(ServerAdapter):
    def run(self, app):
        import socket
        from wsgiref.simple_server import (WSGIRequestHandler, WSGIServer,
                                           make_server)

        ...

        handler_cls = self.options.get("handler_class", FixedHandler)
        server_cls = self.options.get("server_class", WSGIServer)

        ...

        self.srv = make_server(self.host, self.port, app, server_cls, handler_cls)
        self.port = self.srv.server_port
        try:
            self.srv.serve_forever()
        except KeyboardInterrupt:
            self.srv.server_close()
            raise
```

通过适配器模式，可以很方便地修改原有适配或者添加新的 Server 支持。这里不详述了。

# 模板系统

Bottle 实现了自己的模板生成功能，同时也支持主流的 Mako 和 Jinja2。无论使用哪一种，都可以直接调用 `template` 这个简单的接口：

```python
def template(*args, **kwargs):
    tpl = args[0] if args else None
    for dictarg in args[1:]:
        kwargs.update(dictarg)
    adapter = kwargs.pop("template_adapter", SimpleTemplate)
    lookup = kwargs.pop("template_lookup", TEMPLATE_PATH)
    tplid = (id(lookup), tpl)
    if tplid not in TEMPLATES or DEBUG:
        settings = kwargs.pop("template_settings", {})
        if isinstance(tpl, adapter):
            TEMPLATES[tplid] = tpl
            if settings:
                TEMPLATES[tplid].prepare(**settings)
        elif "\n" in tpl or "{" in tpl or "%" in tpl or "$" in tpl:
            TEMPLATES[tplid] = adapter(source=tpl, lookup=lookup, **settings)
        else:
            TEMPLATES[tplid] = adapter(name=tpl, lookup=lookup, **settings)
    if not TEMPLATES[tplid]:
        abort(500, "Template (%s) not found" % tpl)
    return TEMPLATES[tplid].render(kwargs)
```

通过指定 `template_adapter` 使用不同的模板系统，这是很典型的策略模式。另外，这里也同样出现了适配器模式，基于 `BaseTemplate` 来适配各个模板库：

```python
class MakoTemplate(BaseTemplate):
    ...

class Jinja2Template(BaseTemplate):
    ...

class SimpleTemplate(BaseTemplate):
    ...
```

# 自动重启

Bottle 还提供了检测 Python 文件改动并自动重启服务的功能，实现思路也很巧妙：

```python
def run(
    app=None,
    server="wsgiref",
    host="127.0.0.1",
    port=8080,
    interval=1,
    reloader=False,
    quiet=False,
    plugins=None,
    debug=None,
    config=None,
    **kargs
):
    if NORUN:
        return
    if reloader and not os.environ.get("BOTTLE_CHILD"):
        import subprocess

        fd, lockfile = tempfile.mkstemp(prefix="bottle.", suffix=".lock")
        environ = os.environ.copy()
        environ["BOTTLE_CHILD"] = "true"
        environ["BOTTLE_LOCKFILE"] = lockfile
        args = [sys.executable] + sys.argv
        if getattr(sys.modules.get("__main__"), "__package__", None):
            args[1:1] = ["-m", sys.modules["__main__"].__package__]
        # 如果设置了 Reload 那么主进程会执行下面的 try block，
        # 启动子进程并且 Polling 在 while 循环中
        try:
            os.close(fd)  # We never write to this file
            while os.path.exists(lockfile):
                p = subprocess.Popen(args, env=environ)
                while p.poll() is None:
                    os.utime(lockfile, None)  # Tell child we are still alive
                    time.sleep(interval)
                if p.returncode == 3:  # Child wants to be restarted
                    continue
                sys.exit(p.returncode)
        except KeyboardInterrupt:
            pass
        finally:
            if os.path.exists(lockfile):
                os.unlink(lockfile)
        return
    # 如果没有设置 Reload 或者是子进程，则会执行下面的 try block
    # 这时才会真正启动 Server，而且如果是子进程的话还会启动额外的检测线程
    try:
        if debug is not None:
            _debug(debug)
        app = app or default_app()
        if isinstance(app, basestring):
            app = load_app(app)
        if not callable(app):
            raise ValueError("Application is not callable: %r" % app)

        for plugin in plugins or []:
            if isinstance(plugin, basestring):
                plugin = load(plugin)
            app.install(plugin)

        if config:
            app.config.update(config)

        if server in server_names:
            server = server_names.get(server)
        if isinstance(server, basestring):
            server = load(server)
        if isinstance(server, type):
            server = server(host=host, port=port, **kargs)
        if not isinstance(server, ServerAdapter):
            raise ValueError("Unknown or unsupported server: %r" % server)

        server.quiet = server.quiet or quiet
        if not server.quiet:
            _stderr(
                "Bottle v%s server starting up (using %s)..."
                % (__version__, repr(server))
            )
            if server.host.startswith("unix:"):
                _stderr("Listening on %s" % server.host)
            else:
                _stderr("Listening on http://%s:%d/" % (server.host, server.port))
            _stderr("Hit Ctrl-C to quit.\n")
        # 这里判断 Reload 成功就会启动 Daemon thread 检测文件改动
        if reloader:
            lockfile = os.environ.get("BOTTLE_LOCKFILE")
            bgcheck = FileCheckerThread(lockfile, interval)
            with bgcheck:
                server.run(app)
            # 如果文件出现改动则退出子进程
            # 如果是 lockfile 有问题这里的 status 会是 error
            # 子进程也会退出，但 Exit status 不是 3 所以不会重启
            if bgcheck.status == "reload":
                sys.exit(3)
        else:
            server.run(app)
    except KeyboardInterrupt:
        pass
    except (SystemExit, MemoryError):
        raise
    except:
        if not reloader:
            raise
        # 如果是其他的异常，非 quiet 情况下会打印错误栈
        # sleep 之后退出子进程并重启
        if not getattr(server, "quiet", quiet):
            print_exc()
        time.sleep(interval)
        sys.exit(3)

class FileCheckerThread(threading.Thread):
    def __init__(self, lockfile, interval):
        threading.Thread.__init__(self)
        self.daemon = True
        self.lockfile, self.interval = lockfile, interval
        #: Is one of 'reload', 'error' or 'exit'
        self.status = None

    def run(self):
        exists = os.path.exists
        mtime = lambda p: os.stat(p).st_mtime
        files = dict()

        for module in list(sys.modules.values()):
            path = getattr(module, "__file__", "") or ""
            if path[-4:] in (".pyo", ".pyc"):
                path = path[:-1]
            if path and exists(path):
                files[path] = mtime(path)

        while not self.status:
            # 如果 lockfile 不存在或者过于陈旧则通知子进程退出
            # 这里的 interrupt_main 默认会发送给主线程（即子进程）
            # SIGINT 信号，触发 KeyboardInterrupt 异常
            if (
                not exists(self.lockfile)
                or mtime(self.lockfile) < time.time() - self.interval - 5):
                self.status = "error"
                thread.interrupt_main()
            for path, lmtime in list(files.items()):
                if not exists(path) or mtime(path) > lmtime:
                    self.status = "reload"
                    thread.interrupt_main()
                    break
            time.sleep(self.interval)

    def __enter__(self):
        self.start()

    def __exit__(self, exc_type, *_):
        if not self.status:
            self.status = "exit"  # silent exit
        self.join()
        # 这里通过判断类型来 Suppress interrupt_main()
        # 造成的 KeyboardInterrupt 异常，如果 __exit__ 返回 True
        # 则上下文管理器不会抛出异常
        return exc_type is not None and issubclass(exc_type, KeyboardInterrupt)
```

大致的流程图如下：

{{< mermaid theme="dark" >}}
flowchart TD
start(["Start"]) --> reload_or_subproc{"Is reloader set to True\n or is this a child process?"}
reload_or_subproc --> |Yes| subprocess["Spawn a subprocess"]
reload_or_subproc --> |No| reload["Is reloader set to True?"]
subprocess --> reload
reload --> |No| serve["Start serving"]
serve --> if_restart{"Need to restart?"}
if_restart --> |Yes| subprocess
if_restart --> |No| stop(["Stop"])
reload --> |Yes| filechecker["Spawn a file checker thread"]
filechecker --> changes{"Is there any changes\n or broken lockfile?"}
changes --> |Yes| if_restart
changes --> |No| changes
{{</mermaid>}}

其中以 3 作为 Exit status 来标识重启的子进程，其实并不是一种标准用法（也没有固定的标准），可能算作 Python 程序的某种传统吧。

# 扩展性

虽然 Bottle 已经自带了很多常用的工具，比如 Cookie 支持和文件上传，但并不妨碍在其基础上开发扩展，因为有 Hook 和 Plugin。

Hook 类似 Django 的 Middleware，可以在几个固定时机执行特定的功能，比如 `before_request` 和 `after_request`。

Plugin 更灵活一些，也是添加 ORM 等复杂的定制化功能的最好方式。Plugin 基于视图函数执行，既可以全局生效，也能单独进行设置。对此 Bottle 定义了一整套抽象接口，这让 Plugin 不只是函数，也可以定义成复杂的对象，具体参考[官方的开发文档](https://bottlepy.org/docs/dev/plugindev.html)。

# 现状

虽然优点众多，但 Bottle 的没落也不是没有理由的。由于作者坚持单文件模块并且不增加额外依赖，同时又要兼容 Python 2，很大程度上限制框架的发展。不像 Flask，虽然出现得晚，这么多年来也早已发展壮大了。不过我倒是很佩服作者，让 Bottle 保持一直以来的定位：**A fast, simple and lightweight WSGI micro web-framework for Python**。

即使不再流行，Bottle 背后现在仍然有一拨坚定的开发者，这离不开框架本身的实用性和过硬的代码质量。作为源代码学习的项目，Bottle 再合适不过了。而且，如果能够基于一个熟悉程度超高的 Web 框架做开发，体验也会是完全不一样的。

向 Bottle 致敬。

---

*References*

- [Bottle - GitHub](https://github.com/bottlepy/bottle/)

- [Bottle: Python Web Framework](https://bottlepy.org/docs/dev/index.html)

- [Web Frameworks Benchmark](https://web-frameworks-benchmark.netlify.app/)
