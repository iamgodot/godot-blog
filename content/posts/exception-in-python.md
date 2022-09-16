---
title: "Python 中的异常"
date: 2022-09-15T18:02:18+08:00
draft: false
categories:
  - Code
tags:
  - python
keywords:
  - Python 捕获异常
  - Python 自定义异常
  - Python 抛出异常
---

在 Python 中，[EAFP](https://docs.python.org/3/glossary.html#term-EAFP) 的风格很受青睐，这种写法能让代码更加简洁，还可以避免一些重复判断和多线程竞争的问题。为此，了解并熟练使用异常是很重要的。

# 异常类

首先来看一下 Python 内置的[异常类](https://docs.python.org/3/library/exceptions.html#exception-hierarchy)（有省略）：

```tex
BaseException
 +-- SystemExit
 +-- KeyboardInterrupt
 +-- GeneratorExit
 +-- Exception
      +-- StopIteration
      +-- StopAsyncIteration
      +-- ArithmeticError
      +-- AssertionError
      +-- AttributeError
      +-- BufferError
      +-- EOFError
      +-- ImportError
      +-- LookupError
      +-- MemoryError
      +-- NameError
      +-- OSError
      +-- ReferenceError
      +-- RuntimeError
      +-- SyntaxError
      +-- SystemError
      +-- TypeError
      +-- ValueError
      +-- Warning
```

`BaseException` 是所有异常的老祖宗，但很少会用到，通常我们只需要 `Exception`，比如自定义一个错误类型：

```python
# 命名习惯一般以 Error 结尾
class CustomError(Exception):
    def __init__(self, message, status):
        # 这里最好把参数都放进去，之后会统一存在 e.args 中
        super().__init__(message, status)
        self.message = message
        self.status = status
```

除此之外，有两种与 `Exception` 平级的异常需要我们注意，就是 `SystemExit` 和 `KeyboardInterrupt`。

`SystemExit` 可以通过 `sys.exit()` 来触发，会让程序以特定的 Exit code 退出；而 `KeyboardInterrupt` 一般由 `Ctrl-C` 导致，正常情况下也会让 Interpreter 结束工作。

这两种异常都会直接影响程序的执行退出，继承自 `BaseException` 可能也是为了和 `Exception` 做区分。比如 `KeyboardInterrupt`，因为过于不可控（任意发出的打断操作），即使通过异常处理可能也无法正常完成资源回收等操作，所以用注册 Signal handler 的方式来实现 Graceful shutdown 更加合适。

另外，要避免用单独的 `except` 语句，因为默认针对的是 `BaseException`，会把上面两种异常都包含进去。

# 抛出异常

抛出异常是很简单的，基本上只需要记住 `raise` 关键字，后面加上异常对象就好了。

这里有个知识点是 `raise` 的时候也可以直接使用异常类型：

> If it is a class, the exception instance will be obtained when needed by instantiating the class with no arguments.

当然，初始化异常对象并且加入一些上下文信息会更有利于 Debug。

在异常处理中，如果想重新抛出异常，只需要这样做：

```python
def test_exception_chaining():
    try:
        1 / 0
    except ZeroDivisionError as e:
        print('Trying to divide 1 by 0')
        raise e  # 或者直接 raise 即可
```

最后要提的是 [Exception chaining](https://docs.python.org/3/tutorial/errors.html#exception-chaining)，可以理解为一个异常打印消息的优化，在文本中会显示更加清晰的异常链关系：

```python
def test_exception_chaining():
    try:
        1 / 0
    except ZeroDivisionError as e:
        raise ValueError from e
```

此时执行函数打印的错误栈会显示：

```tex
Traceback (most recent call last):
  File "/tmp/test.py", line 3, in test_exception_chaining
    1 / 0
ZeroDivisionError: division by zero

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/tmp/test.py", line 37, in <module>
    foo()
  File "/tmp/test.py", line 5, in test_exception_chaining
    raise ValueError from e
ValueError
```

可以看到，消息中指明了 `ZeroDivisionError` 是导致 `ValueError` 的直接原因。

在这种情况下，`ValueError` 的 `__cause__` 属性会被设置为前面的 `ZeroDivisionError` 对象，而 `__suppress_context__` 为 `True`。

而如果我们不加 `raise ValueError` 后面的 `from e`，则 `__supress_context__` 为 `False`，另一个属性 `__context__` 会被设置为 `ZeroDivisionError` 对象。这是出现连环异常的默认处理，此时错误栈会显示：

`During handling of the above exception, another exception occurred:`

看上去就没有 Exception chaining 那么直观了。

# 异常处理

Python 异常处理语句的完全体如下：

```python
def test_var(var):
    try:
        assert isinstance(var, str)
    except AssertionError:
        print(f'{var} is not a string')
    finally:
        print('End test')
    else:
        print(f'{var} is a string')
```

只有存在至少一个 `except`，才可以使用 `else` 语句。如果不出现任何异常情况，在 `try` 中的代码执行完成之后，会继续执行 `else`。

而 `finally` 很好理解，简单来说就是一定会执行：

> the `finally` clause is executed in any event.

具体可以分为下面几种情况：

- 出现 Unhandle 的异常。
  - 不论是 `except` 没有覆盖还是异常处理过程中又出现了新的异常，都会先完成 `finally` 的逻辑再抛出。
  - 但是如果 `finally` 中直接 `return/break/continue` 了，则异常会被忽略掉。
- 未出现异常或者异常被成功 Handle。
  - 不论是 `try` 还是 `except` 中的 `return/break/continue`，执行之前还是要先处理 `finally` 的逻辑。
  - 同样，如果 `finally` 中直接退出了，那么也不会再继续之前的 `try` 或者 `except`。

总结一下就是 `finally` 最后一定会执行，而且优先级更高。

虽然这套四连很好用，但是代码中写多了比较麻烦，也影响可读性。所幸还有一种更简洁的语法，就是 `with`。

# 使用 with

在上下文管理器协议中，从 `__exit__` 方法的参数中可以得到 `with` 块中执行语句可能抛出的 Exception，所以这里很适合做异常处理。

```python
class ExceptionManager:
    def __init__(self, exception, handler):
        self.exception = exception
        self.handler = handler
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type == self.exception:
            self.handler(exc_val)
            return True
        return False
```

这里要注意 `__exit__` 的返回值，如果布尔结果为 `True`，则异常不会继续抛出。

利用内置的 `contextlib` 还可以这样实现：

```python
from contextlib import contextmanager

@contextmanager
def manage_exception(exception, handler):
    try:
        yield
    except exception as e:
        handler(e)
```

如果想忽略一些特定的异常：

```python
from contextlib import suppress

with suppress(KeyError):
    d = {}
    print(d['foo'])
```

# 一些结论

在使用异常机制时，可以参考下面几点：

1. 提前考虑可能发生的异常，比如非法输入、网络或 I/O 错误等。
2. 对于不合理的情况应当直接抛出异常，交给上层处理。
   1. 如果是底层函数，那么可以直接使用内置异常类型。
   2. 或者自定义内部异常，但是要注意和模块的层级匹配。
3. 异常处理。
   1. 重试。推荐 [Tenacity](https://github.com/jd/tenacity) 这个库。
   2. 记录日志，还可以配合 Sentry 发出告警。
   3. 资源回收与回滚操作。比如关闭打开的文件、回滚数据库等。
4. 返回值。
   1. 直接抛出异常。
   2. 使用默认值，如 `False` 或 `-1`。
   3. 根据异常返回对应的 HTTP Response，这在服务器应用中很常见。
5. 兜底。总会发生意想不到的错误，为了保险起见最好多加一层处理。
   1. 离线任务：做好适当的回滚，不要 Block 其他任务，修复之后再手动触发。
   2. 在线应用：做好记录并返回错误响应，最重要的是不打断程序的正常运行。

---

*References*

- [Errors and Exceptions - Python Docs](https://docs.python.org/3/tutorial/errors.html)
- [Built-in Exceptions - Python Docs](https://docs.python.org/3/library/exceptions.html)
- [LBYL vs EAFP - RealPython](https://realpython.com/python-lbyl-vs-eafp/)
- [Python 工匠： 异常处理的三个好习惯](https://www.zlovezl.cn/articles/three-rituals-of-exceptions-handling/)
