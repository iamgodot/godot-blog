---
title: "Python Logging 源码分析"
date: 2022-04-09T09:52:51+08:00
draft: true
mermaid: true
categories:
  - Code
tags:
  - python
keywords:
  - Python Logging
  - Logging 源码
  - Logging 用法详解
---

阅读了源码之后，我对 Python Logging 模块的几大疑惑都得到了解答：

- 为什么 Logger 和 Handler 都有 `setLevel` 方法？
- Logging 中会出现 Race condition 吗？（感觉都是很直接的 write 操作）

- 正式环境中想看日志又没办法动态调整 `logLevel`，感觉很鸡肋。

- 用起来好像还不如 `print` 方便。
- 会有性能问题吗？

# 日常使用

首先要了解下 Logging 的用法。

## 1. 配置

基本上有三种方式，代码、文件和字典。先看下如何用代码设置：

```python
import logging

# create logger
logger = logging.getLogger('simple_example')
logger.setLevel(logging.DEBUG)

# create console handler and set level to debug
ch = logging.StreamHandler()
ch.setLevel(logging.DEBUG)

# create formatter
formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

# add formatter to ch
ch.setFormatter(formatter)

# add ch to logger
logger.addHandler(ch)
```

Logging 的接口很多都是 `camelCase` 而非 `snake_case`，应该有历史原因。另外注意 `setFormatter` 和 `addHandler` 这种用词的区别，说明 `Handler` 的 `Formatter` 只能有一个，而 `Logger` 可以添加多个 `Handler`。

第二种是文件，使用 `fileConfig` 接口读取，这里要求文件使用符合 [ConfigParser](https://docs.python.org/3/library/configparser.html#module-configparser) 的格式。

最后是利用 `dictConfig` 来读取一个字典，这就提供了使用 `json/yaml` 作为配置文件的可能，也是比较常用的一种方式。

当然如果不做任何自定义配置也是可以 Logging 的，这时默认会将日志输出到 `sys.stderr` 并且 `logLevel` 为 `WARNING`。

## 2. 日志打印

像 `debug/info/warning/error` 这些方法都会创建对应 `logLevel` 的日志对象，不过 `exception` 特殊一点，相当于 `error`，但是会打印额外的异常信息。

Logging 本身的操作一般不会导致异常，但如果出现（比如配置错误或者打日志方法传入的字符串 格式化出错），除了 `SystemExit` 和 `KeyboardInterrupt` 之外，都会由 `Handler` 来处理：默认不会抛出，而是将错误打印到 `sys.stderr`。这种行为是由 `logging.raiseExceptions` 来控制的，也可以设置为 False，这样就会把异常直接吞掉。

## 3. 更多用法

最后还有一些进阶的用法，比如自定义日志对象，可以通过定制的 `Filter` 添加额外的日志信息：

```python
import logging
from random import choice

class ContextFilter(logging.Filter):
    """
    This is a filter which injects contextual information into the log.

    Rather than use actual contextual information, we just use random
    data in this demo.
    """

    USERS = ['jim', 'fred', 'sheila']
    IPS = ['123.231.231.123', '127.0.0.1', '192.168.0.1']

    def filter(self, record):

        record.ip = choice(ContextFilter.IPS)
        record.user = choice(ContextFilter.USERS)
        return True
```

# 代码分析

通过画图可以快速熟悉代码中的设计，比如用流程图来展示 Logging 的执行过程：

{{< mermaid theme="dark" >}}
flowchart TB
  Start(["Start Logging call e.g. Logger.info(...)"]) --> LoggerLevel{"Is current\n level disabled?"}
  LoggerLevel --> |No| Stop(["Stop"])
  LoggerLevel --> |Yes| LogRecord["Create log record"]
  LogRecord --> Filter{"Should the record\n be filtered?"}
  Filter --> |Yes| Stop
  Filter --> |No| Handler

  subgraph Handler["Handle for current logger"]
    Lock["Acquire RLock"] --> Emit["Format log and write to stream"]
  end

  Handler --> Propagate{"Should propagate\n for current logger?"}
  Propagate --> |No| Stop
  Propagate --> |Yes| ParentLogger{"Is there a\n parent logger?"}
  ParentLogger --> |No| Stop
  ParentLogger --> |Yes| SetCurrentLogger["Set parent logger as current"]
  SetCurrentLogger --> Handler
{{</mermaid>}}

用类图表达各个 Class 之间的关系：

{{< mermaid theme="dark" >}}

classDiagram
    direction TB
    Filterer <|-- Logger
    Filterer <|-- Handler
    LoggerAdapter "1" *-- "1" Logger
    Manager "1" *-- "1..*" Logger
    Logger <|-- RootLogger
    Logger "1" *-- "*" Filter
    Logger "1" *-- "*" Handler
    Handler "1" *-- "1" Formatter
    Handler <-- LogRecord
    Handler <|-- StreamHandler
    StreamHandler <|-- FileHandler
    Formatter <-- LogRecord
    Filter <-- LogRecord
    class Manager{
        +Logger root
        +Int disable
        +Dict loggerDict
        +getLogger()
        -_fixupParent()
        -_fixupChildren()
        -_clear_cache()
    }
    class LoggerAdapter{
        +Logger logger
        +process()
        +debug/info/warning/error/...()
    }
    class Filterer{
        +List filters
        +addFilter()
        +removeFilter()
        +filter()
    }
    class Logger{
        +debug/info/warning/error/...()
        +isEnabledFor()
        -_log()
        +makeRecord()
        +handle()
        +addHandler()
    }
    class Handler{
        -Str _name
        +Formatter formatter
        +RLock lock
        +acquire()
        +release()
        +handle()
        +emit()
        +flush()
        +handleError()
    }
    class StreamHandler{
        +stream
        +setStream()
    }
    class FileHandler{
        +filename
        +name
        +encoding
    }
    class Filter{
        +Str name
        +filter()
    }
    class Formatter{
        -_fmt
        -_style
        +datefmt
        +format()
        +formatTime()
        +formatMessage()
        +formatException()
        +formatStack()
    }
    class LogRecord{
        +Str name
        +Str msg
        +args
        +getMessage()
    }

{{</mermaid>}}

至此可以比较清晰地了解日志过程中 Logger/Filter/Handler/Formatter 几个组件之间的交互了。Logging 的实现非常 OOP，但并不是很 Pythonic，比如一些不必要的 Setter/Getter、本应该却没有使用 `with` 的地方（大量线程锁的获取和释放）、`camelCase` 的函数命名等等。不过鉴于这个模块出现得很早，可能也背了不少历史包袱吧。

## 1. 线程安全

模块中有不少全局的数据结构变量，这也解释了为什么要保证线程安全，比如 `logLevel`：

```python
CRITICAL = 50
FATAL = CRITICAL
ERROR = 40
WARNING = 30
WARN = WARNING
INFO = 20
DEBUG = 10
NOTSET = 0

_levelToName = {
    CRITICAL: 'CRITICAL',
    ERROR: 'ERROR',
    WARNING: 'WARNING',
    INFO: 'INFO',
    DEBUG: 'DEBUG',
    NOTSET: 'NOTSET',
}
_nameToLevel = {
    'CRITICAL': CRITICAL,
    'FATAL': FATAL,
    'ERROR': ERROR,
    'WARN': WARNING,
    'WARNING': WARNING,
    'INFO': INFO,
    'DEBUG': DEBUG,
    'NOTSET': NOTSET,
}
```

还有 `Handlers`：

```python
_handlers = weakref.WeakValueDictionary()  #map of handler names to handlers
_handlerList = [] # added to allow handlers to be removed in reverse of order initialized
```

线程锁的使用主要在两处，一是全局范围的，保证像上面这两种全局变量的安全读写：

```python
#_lock is used to serialize access to shared data structures in this module.
#This needs to be an RLock because fileConfig() creates and configures
#Handlers, and so might arbitrary user threads. Since Handler code updates the
#shared dictionary _handlers, it needs to acquire the lock. But if configuring,
#the lock would already have been acquired - so we need an RLock.
#The same argument applies to Loggers and Manager.loggerDict.
#
_lock = threading.RLock()

def _acquireLock():
    if _lock:
        _lock.acquire()

def _releaseLock():
    if _lock:
        _lock.release()
```

如注释所说，因为 `fileConfig()` 时会重复上锁，需要 Re-entrant lock；另外一处在 `Handler` 内部：

```python
class Handler(Filterer):
    def createLock(self):
        """
        Acquire a thread lock for serializing access to the underlying I/O.
        """
        self.lock = threading.RLock()
        _register_at_fork_reinit_lock(self)

    def handle(self, record):
        rv = self.filter(record)
        if rv:
            self.acquire()
            try:
                self.emit(record)
            finally:
                self.release()
        return rv
```

很明显，这里是为了保证 `I/O` 操作的原子性而上锁，不过似乎用普通的 `Mutex` 也是可以的。需要注意的是，这里的原子操作只针对一个线程 + 一个文件描述符的场景，如果有两个线程分别打开同一个文件日志的话是存在乱写的可能的，也就是 `garble`。同理，在多进程下 Logging 是不安全的，比较保险的做法是使用额外的全局锁（效率低）或者 `QueueHandler`。其实如果不写入文件直接输出到 `sys.stderr` 问题并不大，即使出现 `garble`（概率很低）影响也很小，在应用容器之外再做日志收集和聚合就好了。

为了尽量保证 `I/O` 不出现 `garble`，`Handler` 也尽量做了优化，比如 `StreamHandler` 的 `emit` 方法：

```python
class StreamHandler(Handler):
    def emit(self, record):
        try:
            msg = self.format(record)
            stream = self.stream
            # issue 35046: merged two stream.writes into one.
            stream.write(msg + self.terminator)  # 使用一次 write 操作
            self.flush()  # 及时清空 buffer 内容
        except RecursionError:  # See issue 36272
            raise
        except Exception:
            self.handleError(record)
```

## 2. Logger 结构

Logging 对于 Loggers 的结构设计有点类似前缀树。首先是存在一个 Root logger 作为根节点的，这也是为什么可以直接用 `import logging;logging.info(...)`；其次 `name` 不为空的 Logger 会按照 `.` 来切割，比如 `a.b.c` 这个 Logger 可以被划分为三层，`a` 和 `a.b` 这两个 Logger 如果存在的话则作为 `a.b.c` 的 parent，如果不存在则初始化为 `PlaceHolder` 占位；最后，每定义一个 Logger 都会创建对应的节点并更新上下游的父子节点。这些逻辑都被封装在 `Manager` 类中，简单看下代码（这里也可以看出全局线程锁的应用）：

```python
class Manager(object):
    def getLogger(self, name):
        rv = None
        if not isinstance(name, str):
            raise TypeError('A logger name must be a string')
        _acquireLock()
        try:
            if name in self.loggerDict:
                rv = self.loggerDict[name]
                if isinstance(rv, PlaceHolder):
                    ph = rv
                    rv = (self.loggerClass or _loggerClass)(name)
                    rv.manager = self
                    self.loggerDict[name] = rv
                    self._fixupChildren(ph, rv)
                    self._fixupParents(rv)
            else:
                rv = (self.loggerClass or _loggerClass)(name)
                rv.manager = self
                self.loggerDict[name] = rv
                self._fixupParents(rv)
        finally:
            _releaseLock()
        return rv

    # 这两个方法会在创建 Logger 时更新相关的父子节点
    def _fixupParents(self, alogger):
        """
        Ensure that there are either loggers or placeholders all the way
        from the specified logger to the root of the logger hierarchy.
        """
        name = alogger.name
        i = name.rfind(".")
        rv = None
        while (i > 0) and not rv:
            substr = name[:i]
            if substr not in self.loggerDict:
                self.loggerDict[substr] = PlaceHolder(alogger)
            else:
                obj = self.loggerDict[substr]
                if isinstance(obj, Logger):
                    rv = obj
                else:
                    assert isinstance(obj, PlaceHolder)
                    obj.append(alogger)
            i = name.rfind(".", 0, i - 1)
        if not rv:
            rv = self.root
        alogger.parent = rv

    def _fixupChildren(self, ph, alogger):
        """
        Ensure that children of the placeholder ph are connected to the
        specified logger.
        """
        name = alogger.name
        namelen = len(name)
        for c in ph.loggerMap.keys():
            #The if means ... if not c.parent.name.startswith(nm)
            if c.parent.name[:namelen] != name:
                alogger.parent = c.parent
                c.parent = alogger
```

像 `logging.getLogger` 实际上就是用的这里提供的接口，除此之外在 `propagate` 的时候也是通过 `Manager` 来定位 Parent logger。

## 3. 设置 logging level

`Handler` 的 `setLevel` 其实没什么用，不像 `Logger` 会使用 `logLevel` 来辅助判断是否 `enabled`，`Handler` 最多在 `repr` 时打印一下，不过这样设计又会造成两者的 `logLevel` 不一致，总之使用时以 `Logger` 为准就好了。

另外其实在 Logging 的设计中 `logLevel` 虽然有预设，但是是可以自定义的。大概可能是默认设置已经很够用了，还没见到过哪里真的去做定制的。

最后，不得不说，Logging 的注释非常详细，甚至有过度解释代码的嫌疑，不过作为这么古老的基础模块，可能也不是一件坏事。

# 一些结论

除了代码，Logging 的官方文档也介绍了许多场景下的 Best practice。

如果想在应用运行过程中更改配置，那么一种简单的实现是使用 `fileConfig` 并开启额外的端口来监听文件内容的变化。当然如果从更高层的角度来考虑，解决办法还有很多，比如在 Redis 中保存配置，再单独启动一个 Worker 来订阅更新并在变动时重新设置。

还有关于[性能的优化](https://docs.python.org/3/howto/logging.html#optimization)。另外注意不要初始化太多的 Logger，因为这些实例是不会自动 GC 的（正常来说按模块来初始化就够了，比如`logger = logging.getLogger(__name__)`，这样也正好合理利用了模块的名称来划分 Logger 层级）。如果有需要的话，可以考虑使用 `LoggerAdapter` 来维护额外的上下文信息。

最后，到底选择 Logging 还是 `print`？除了[这里](https://docs.python-guide.org/writing/logging/#or-print)提到的，如果涉及到多线程的场景，前者会更安全一些。而 `print` 简单好用，还不需要额外 `import`。所以很简单，一切选择都应当以实际场景为准，没有绝对的好坏和对错。

---

*References*

- [Logging HOWTO - Python Docs](https://docs.python.org/3/howto/logging.html)
- [Logging Cookbook - Python Docs](https://docs.python.org/3/howto/logging-cookbook.html)
- [Logging - The Hitchhiker's Guide to Python](https://docs.python-guide.org/writing/logging/)
