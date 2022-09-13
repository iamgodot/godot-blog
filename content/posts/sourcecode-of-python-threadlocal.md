---
title: "Python 中的 TLS 是如何实现的"
date: 2022-04-11T16:13:49+08:00
draft: false
categories:
  - Code
tags:
  - python
keywords:
  - Python 的 Threadlocal
  - Python 的 threading.local 实现
  - Flask 的 Local
---

TLS(Thread Local Storage)，或者说 Threadlocal，可以说是一种并发编程的常用模式，既实现了线程之间的资源隔离，又满足了全局变量的使用。

从 TLS 出发，这篇文章研究了 Python 中的 Threadlocal 是如何实现的，比如自带的 `threading.local`，再比如 Flask 框架中 `Local` 对象。

# Why Threadlocal

先思考一下为什么要用 Threadlocal，这就不得不提到线程安全。Race condition 说到底是因为数据共享和非原子操作，这可以体现在函数的两种基本写法：一种是显式地传参（参数对象也可能变化？这也是为什么最好不要传递可变对象），没有共享自然安全；另一种就是全局对象，这么写既简化了函数签名，代码也比较清晰，缺点就是很容易出现线程不安全的问题，所以经常会和锁配合使用。

而 Threadlocal 就结合了两者的优点，在共享全局变量的同时，保证每个线程操作的都是自己独有的数据对象。

对比一下 Django 和 Flask 两大框架就会发现，前者总是在视图函数中显式声明 `request` 参数，而后者的只需要 import 一次就可以到处使用。在 Flask 的[文档](https://flask.palletsprojects.com/en/2.0.x/advanced_foreword/)中，Armin Ronacher 也提及了这一点：

> For example, Flask uses thread-local objects internally so that you don’t have to pass objects around from function to function within a request in order to stay threadsafe.

不过 Flask 并没有直接使用 Python 内置的 `threading.local`，而是重新做了实现。

# Threading.local

Python 的`threading.local` 用法很简单：

```python
from threading import Thread, local

mydata = local()
mydata.number = 42
print(mydata.number)  # 42

nums = []
def f():
    mydata.number = 11
    nums.append(mydata.number)
    
t = Thread(target=f)
t.start()
t.join()
print(nums)  # [11]
print(mydata.number)  # 42
```

线程可以共享全局对象，但实际上各自维护了数据，互不影响。如何做到的呢？其实思路也很简单，在字典中给每个线程都创建单独的字典，在使用之前先切换过去，看上去好像在用同一个变量，其实内部分得很清楚。下面看看源码：

```python
...

@contextmanager
def _patch(self):
    ...
    with impl.locallock:
        object.__setattr__(self, "__dict__", dct)
        yield

class local:
	...   
    def __getattribute__(self, name):
        with _patch(self):
            return object.__getattribute__(self, name)

    def __setattr__(self, name, value):
        if name == "__dict__":
            raise AttributeError(
                "%r object attribute '__dict__' is read-only" % self.__class__.__name__
            )
        with _patch(self):
            return object.__setattr__(self, name, value)

...
```

其中 `local` 在 `get/set` 的时候都偷偷地 `_patch` 了下，那么 `_patch` 又做了什么呢？原来是临时把 `local` 的 `__dict__` 替换成了另外的从 `impl` 取到的 dict。其实现在已经大概能猜到了，通过把 local 的属性字典替换成当前线程自己的字典，就实现了 Threadlocal 的核心功能。现在来看 `impl` 的内部结构：

```python
class _localimpl:
    __slots__ = "key", "dicts", "localargs", "locallock", "__weakref__"

    def __init__(self):
        self.key = "_threading_local._localimpl." + str(id(self))
        # { id(Thread) -> (ref(Thread), thread-local dict) }
        self.dicts = {}

    def get_dict(self):
        thread = current_thread()
        return self.dicts[id(thread)][1]

    def create_dict(self):
        localdict = {}
        key = self.key
        thread = current_thread()
        idt = id(thread)

        def local_deleted(_, key=key):
            thread = wrthread()
            if thread is not None:
                del thread.__dict__[key]

        def thread_deleted(_, idt=idt):
            local = wrlocal()
            if local is not None:
                dct = local.dicts.pop(idt)

        wrlocal = ref(self, local_deleted)
        wrthread = ref(thread, thread_deleted)
        thread.__dict__[key] = wrlocal
        self.dicts[idt] = wrthread, localdict
        return localdict
```

从第六行的注释可以看出，`dicts` 中保存了每个线程及其对应的字典，具体是用线程 id 对应到一个包含了线程弱引用和字典的元组。此外还有个单独的 `self.key` 属性，从 `create_dict` 方法中发现是为了给线程设置并标识 `impl` 本身用的。另外 `wrlocal` 和 `wrthread` 都小心地用了弱引用，这样就不影响 `impl` 和 `thread` 的内存回收，同时还会从相关字典中删除已经回收的对象（`local_deleted` 和 `thread_deleted` 作为 `weakref` 的 callback 函数）。

现在再回头看看 `local` 其余的部分：

```python
class local:
    __slots__ = "_local__impl", "__dict__"

    def __new__(cls, /, *args, **kw):
        if (args or kw) and (cls.__init__ is object.__init__):
            raise TypeError("Initialization arguments are not supported")
        self = object.__new__(cls)
        impl = _localimpl()
        impl.localargs = (args, kw)
        impl.locallock = RLock()
        object.__setattr__(self, "_local__impl", impl)
        impl.create_dict()
        return self
```

有一点不太好理解，就是为什么要改写 `__new__`，而且上来就做了一个异常判断，为什么不在 `__init__` 中实现呢？其实这是为了支持以继承的方式来定制 `local`：

```python
class MyLocal(local):
    def __init__(self, /, **kw):
        self.__dict__.update(kw)
        
mydata = MyLocal(color='red')
```

在 `__new__` 里 hack 让继承覆盖 `__init__` 少了很多负担，又因为 `local` 本身是不支持参数的，所以有了一开始的异常判断。

接下来说说 `impl` 的 `localargs` 和 `locallock`：

```python
@contextmanager
def _patch(self):
    impl = object.__getattribute__(self, "_local__impl")
    try:
        dct = impl.get_dict()
    except KeyError:
        dct = impl.create_dict()
        args, kw = impl.localargs
        self.__init__(*args, **kw)
    with impl.locallock:
        object.__setattr__(self, "__dict__", dct)
        yield
```

一目了然，`localargs` 是在为新线程创建字典时重新初始化（对 Subclass 的情况很必要），而 `locallock` 自然是要保证线程安全，因为这里的 `__setattr__` 与后面 `local` 的 `get/set` 操作中间可能会出现 Race condition。

到这里代码已经分析得差不多了，不过还要解释下 `__slots__`。`local` 和 `_localimpl` 中都定义了 `__slots__` 来限制可用属性，既可以优化内存也能保证使用安全。同时为了不影响弱引用和属性赋值，它们又在各自的 `__slots__` 中分别加入了 `__weakref__` 和 `__dict__`。当然，这对 `local` 的使用也产生了负影响：

```python
class MyLocal(local):
    __slots__ = 'number'
    
mydata = MyLocal()
mydata.number = 42  # 这里的 number 是所有线程共享的
```

这是因为 `__slots__` 中的属性是由数据修饰符来控制的，不在 `__dict__` 中保存，因此 `_patch` 无法产生效果。

# Context Locals

现在要说说 Flask 使用 Threadlocal 的思路，前面提到，这样的好处是不用到处传参，但必须要保证线程安全。然而对于一个 Web 框架来说，还需要考虑更多：

> This approach, however, has a few disadvantages. For example, besides threads, there are other types of concurrency in Python. A very popular one is greenlets. Also, whether every request gets its own thread is not guaranteed in WSGI. It could be that a request is reusing a thread from a previous request, and hence data is left over in the thread local object.

除了 `greenlet`，同一个线程也可能被用于处理多个请求，那么 `threading.local` 就不够用了，Flask 也因此重新做了实现（其实是 Werkzeug）。下面看一下源码：

```python
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ("__storage__", "__ident_func__")

    def __init__(self):
        object.__setattr__(self, "__storage__", {})
        object.__setattr__(self, "__ident_func__", get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
```

相比 `threading.local` 这个版本简单了不少，首先尝试 import `greenlet` 的 `get_ident` 方法作为内部的 `__ident_func__`，如果失败再 fallback 到线程上面，这就解决了 `greenlet` 的问题。而对于请求复用线程的情况还引入了 `LocalManager`：

```python
class LocalManager(object):
    def __init__(self, locals=None, ident_func=None):
        if locals is None:
            self.locals = []
        elif isinstance(locals, Local):
            self.locals = [locals]
        else:
            self.locals = list(locals)
        if ident_func is not None:
            self.ident_func = ident_func
            for local in self.locals:
                object.__setattr__(local, "__ident_func__", ident_func)
        else:
            self.ident_func = get_ident

    def get_ident(self):
        return self.ident_func()

    def cleanup(self):
        for local in self.locals:
            release_local(local)

    def make_middleware(self, app):
        def application(environ, start_response):
            return ClosingIterator(app(environ, start_response), self.cleanup)

        return application
```

简单来说就是线程在处理完 WSGI 请求之后会调用 `cleanup` 方法来保证 `release_local` 的执行，这样会把之前的数据字典从 `local` 中删除，然后在下一个请求中重新创建。

相比 `threading.local` 这种实现更直接明了，当然前者也是为了支持 Subclass 等功能和更底层的优化，归根到底是由不同的需求决定的。

# Context Variable

除了线程和 `greenlet`，还有协程，如此一来上面那一套也不够用了。怎么办？Python 3.7 推出了 [contextvars](https://docs.python.org/3/library/contextvars.html) 这个库来保证异步任务执行的上下文隔离，不管是线程还是协程都可以直接用它做到类似 Threadlocal 的事情。

所以 Werkzeug 从 2.x 版本开始也使用 `ContextVar` 来实现 `Local` 了：

```python
from contextvars import ContextVar

class Local:
    __slots__ = ("_storage",)

    def __init__(self) -> None:
        object.__setattr__(self, "_storage", ContextVar("local_storage"))

    def __iter__(self) -> t.Iterator[t.Tuple[int, t.Any]]:
        return iter(self._storage.get({}).items())

    def __call__(self, proxy: str) -> "LocalProxy":
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    def __release_local__(self) -> None:
        self._storage.set({})

    def __getattr__(self, name: str) -> t.Any:
        values = self._storage.get({})
        try:
            return values[name]
        except KeyError:
            raise AttributeError(name) from None

    def __setattr__(self, name: str, value: t.Any) -> None:
        values = self._storage.get({}).copy()
        values[name] = value
        self._storage.set(values)

    def __delattr__(self, name: str) -> None:
        values = self._storage.get({}).copy()
        try:
            del values[name]
            self._storage.set(values)
        except KeyError:
            raise AttributeError(name) from None
```

代码相比之前更加简洁，而且外部接口完全不变。

# The End

可以想象，Threadlocal 的写法是从线程安全和代码简洁等需求中演变而来的。理解了这些，就能更好地在设计开发中做出正确的选择。

---

- [Thread-Locals in Flask](https://flask.palletsprojects.com/en/2.1.x/advanced_foreword/)
- [Context Locals - Werkzeug](https://werkzeug.palletsprojects.com/en/2.1.x/local/)
- [Contextvars - Python Docs](https://docs.python.org/3/library/contextvars.html)
