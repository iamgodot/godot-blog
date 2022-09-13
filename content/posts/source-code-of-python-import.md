---
title: "Python Import 源码阅读"
date: 2022-09-10T18:20:40+08:00
draft: false
categories:
  - Code
tags:
  - python
keywords:
  - Python import 用法
  - Python import 源码
---

浅析一下 Python 中的 Import 机制。代码版本：`3.11.0 - 4dd8219`。

# Module & Package

Import 的对象就是各种各样的模块，即 Module。如何定义呢？官方文档这样描述：

> A module is a file containing Python definitions and statements.

实际上，Module 可以是 Builtin，也可以是 C extension，但最常见的存在形式还是 `*.py` 文件。

一个 Module 同时也可能是 Package，此时它从文件升级为目录，就可以拥有 Submodule 了。Package 分为两种：

1. Regular package：这类 Package 必须包含一个 `__init__.py` 文件，代表了这个 Module(Package) 本身。
2. Namespace package：如果不存在 `__init__.py`，Python 会将其创建为此类 Package。Namespace package 的特殊之处在于同名的 Package 可以出现在多个目录下，而 Import 完成之后又可以统一使用。

不管是哪一种 Package，都会有 `__path__` 属性，指向目录的路径。属性值是个列表，这对于 Namespace package 尤为重要。除此之外，一个 Module 还有：

- `__name__`：模块名称。
- `__file__`：文件位置。
- `__package__`：主要是为了在 Relative import 时计算起点位置。
  - 如果是 Package 则设置为 `__name__`。
  - 如果非 Package 则设置为 Parent package 的名称（Top-level 的 Module 应为空字符串）。
  - 如果以脚本执行，那么取值为 `None`。

# How to Import

一般情况下 Import 都是通过 `import` 关键字完成的，可分为两大类：

- Absolute import：`from foo import bar as bat`
  - 其中 `bar` 可以是 Function，Class 或者 Submodule。
  - 如果 `import module.submodule`，则实际 Import 的是整个 `module`。
  - Wildcard import：`from module import *`
    - 如果 `module` 中定义了 `__all__`，则只会 Import 其指定的部分。
    - 如果没有 `__all__`，那么非 `_` 开头的属性都会被 Import。
- Relative import：`from ..foo import bar`
  - 这种方式会以当前 Module 为起点定位 `foo`，一般会出现两种错误：
    - 相对位置超出了 Top-level package：`ImportError: attempted relative import beyond top-level package`
    - 以脚本的方式运行：`ImportError: attempted relative import with no known parent package`
  - 以模块的方式运行文件：`python -m package.module`
    - 此时 `module` 的 `__name__` 仍然是 `__main__`，和脚本方式一样。
    - 而 `__package__` 则不会是 `None`，而是 `package`。

此外，Python 还提供了 `importlib`，这样就可以在代码中动态 Import。实际上，`__import__`（也就是 `import` 关键字） 和 `importlib.import_module` 都是基于同一套 Codebase 实现的。

# How Import Works

先来看两个接口的实现：

```python
def __import__(name, globals=None, locals=None, fromlist=(), level=0):
    if level == 0:
        module = _gcd_import(name)
    else:
        globals_ = globals if globals is not None else {}
        package = _calc___package__(globals_)
        module = _gcd_import(name, package, level)
    if not fromlist:
        if level == 0:
            return _gcd_import(name.partition('.')[0])
        elif not name:
            return module
        else:
            cut_off = len(name) - len(name.partition('.')[0])
            return sys.modules[module.__name__[:len(module.__name__)-cut_off]]
    elif hasattr(module, '__path__'):
        return _handle_fromlist(module, fromlist, _gcd_import)
    else:
        return module
```

```python
def import_module(name, package=None):
    level = 0
    if name.startswith('.'):
        if not package:
            msg = ("the 'package' argument is required to perform a relative "
                   "import for {!r}")
            raise TypeError(msg.format(name))
        for character in name:
            if character != '.':
                break
            level += 1
    return _bootstrap._gcd_import(name[level:], package, level)
```

可以发现，两者在实际 Import 时调用的都是 `_gcd_import` 这个方法。`gcd` 的意思是 `greatest common denominator`，表示其作为两个接口的主要功能的公共函数。

相对来说，`__import__` 要更灵活一点，因为还提供了 `fromlist` 参数，可以指定要 Import 的具体属性，在代码中会对其做进一步的解析和校验。

下面从 `_gcd_import` 出发，梳理一下 Import 的具体流程：

![](https://static.iamgodot.com/content/images/20220911182414.png)

一开始的 Sanity check 和 Name resolving 主要是针对 Relative import 的情况，比较容易理解。接下来会在 `sys.modules` 中检查 Module 是否已经存在，避免重复开销。如果 Parent module 尚不存在，则以递归的方式先对其做 Import，再继续执行当前流程。

正式的 Import 主要包括 Find 和 Load 两大步骤。在大多数情况下，都是先定位到对应的 `*.py` 文件，读取到内存，最后生成一个 Module 对象并返回。

看上去很简单，但有几个问题要考虑：

1. 在多线程情况下共享 `sys.modules` 这个字典结构，必须保证操作的原子性，还要考虑到 Deadlock。
2. 模块文件可能分散在系统的多个目录下，因此必须要实现一个有效的管理和搜索策略。
3. Import 作为底层功能，还牵涉到很多 I/O 操作，所以在速度上要尽可能地优化。

先看多线程的处理。这里引入了 Global import lock 和 Module lock 两种锁。

```python
# A dict mapping module names to weakrefs of _ModuleLock instances
# Dictionary protected by the global import lock
_module_locks = {}

class _ModuleLockManager:
    def __init__(self, name):
        self._name = name
        self._lock = None

    def __enter__(self):
        self._lock = _get_module_lock(self._name)
        self._lock.acquire()

    def __exit__(self, *args, **kwargs):
        self._lock.release()

def _get_module_lock(name):
    # 获取 Global lock
    _imp.acquire_lock()
    try:
        try:
            # 尝试获取 Module lock
            # 所有的 Module lock 都在一个字典中维护
            lock = _module_locks[name]()
        except KeyError:
            lock = None

        if lock is None:
            if _thread is None:
                lock = _DummyModuleLock(name)
            else:
                lock = _ModuleLock(name)
            # 这里通过 weakref 的 callback 避免内存泄漏
            def cb(ref, name=name):
                _imp.acquire_lock()
                try:
                    if _module_locks.get(name) is ref:
                        del _module_locks[name]
                finally:
                    _imp.release_lock()

            _module_locks[name] = _weakref.ref(lock, cb)
    finally:
        _imp.release_lock()

    return lock
```

```python
class _ModuleLock:
    def __init__(self, name):
        self.lock = _thread.allocate_lock()
        self.wakeup = _thread.allocate_lock()
        self.name = name
        self.owner = None
        self.count = 0
        self.waiters = 0

    def has_deadlock(self):
        # Deadlock avoidance for concurrent circular imports.
        me = _thread.get_ident()
        tid = self.owner
        seen = set()
        while True:
            lock = _blocking_on.get(tid)
            if lock is None:
                return False
            tid = lock.owner
            # 如果我等待的锁的 owner 在等待我手上的锁，说明出现 Deadlock
            if tid == me:
                return True
            if tid in seen:
                return False
            seen.add(tid)

    def acquire(self):
        ...

    def release(self):
        ...
```

下面来看 Module lock 的使用：

```python
_NEEDS_LOADING = object()  # 作为 Sentinel object

def _find_and_load(name, import_):
    module = sys.modules.get(name, _NEEDS_LOADING)
    if (module is _NEEDS_LOADING or
        getattr(getattr(module, "__spec__", None), "_initializing", False)):
        # 这里先获取 Global lock 和 Module lock
        with _ModuleLockManager(name):
            module = sys.modules.get(name, _NEEDS_LOADING)
            if module is _NEEDS_LOADING:
                # 开始真正的 Import
                return _find_and_load_unlocked(name, import_)
        # 如果 _initializing 为 True，说明有其他 Thread 正在 Import
        _lock_unlock_module(name)

    if module is None:
        message = ('import of {} halted; '
                   'None in sys.modules'.format(name))
        raise ModuleNotFoundError(message, name=name)

    return module

...

def _lock_unlock_module(name):
    # 两个主要作用
    # 一是通过 lock 的 acquire&release 来保证 Initializing 的完成
    lock = _get_module_lock(name)
    try:
        lock.acquire()
    # 二是捕捉并忽略 Deadlock 异常
    except _DeadlockError:
        pass
    else:
        lock.release()
```

获取 Module lock 之后，下一步要找到 Spec，它决定了 Module 的类别与一些相关属性，以及后面加载要使用的 Loader。为了查找 Spec，解释器会分别尝试下面三种 Importer 的 `find_spec` 方法，这些 Importer 是在 `sys.meta_path` 中定义的：

1. `BuiltinImporter`
2. `FrozenImporter`
3. `PathFinder`

前两者针对的是预编译好的模块，最常用的其实是 `PathFinder`，因为 Python 自带的函数库和第三方库都要通过它找到模块文件再进行加载。这些文件分散在系统的多个路径下，为了方便管理，`sys.path` 中定义了一个目录列表，里面每个条目都代表一个特定的搜索路径。对于每个条目，`PathFinder` 会调用对应的 `PathEntryFinder`(其实是 `FileFinder`) 进行查找。

因为目录可能很多，所以这里又引入了缓存优化（前面的 `sys.modules` 也是一层缓存）：先在 `sys.path_importer_cache` 中寻找 `PathEntryFinder`，如果没有，再调用 `sys.path_hooks` 中的钩子函数做初始化创建并更新到 `cache` 中。

针对 Namespace package 的处理也在这部分：

```python
class PathFinder:
    ...
    
    @classmethod
    def _get_spec(cls, fullname, path, target=None):
        namespace_path = []
        # 遍历 sys.path 中的每个条目
        for entry in path:
            if not isinstance(entry, (str, bytes)):
                continue
            # 优先从 sys.path_importer_cache 中查找
            # 如果没有则通过 sys.path_hooks 生成
            finder = cls._path_importer_cache(entry)
            if finder is not None:
                if hasattr(finder, 'find_spec'):
                    spec = finder.find_spec(fullname, target)
                else:
                    spec = cls._legacy_get_spec(fullname, finder)
                if spec is None:
                    continue
                if spec.loader is not None:
                    return spec
                portions = spec.submodule_search_locations
                if portions is None:
                    raise ImportError('spec missing loader')
                # 记录所有可能作为 Namespace package 的 path 目录
                namespace_path.extend(portions)
        else:
            spec = _bootstrap.ModuleSpec(fullname, None)
            spec.submodule_search_locations = namespace_path
            return spec

    @classmethod
    def find_spec(cls, fullname, path=None, target=None):
        if path is None:
            path = sys.path
        spec = cls._get_spec(fullname, path, target)
        if spec is None:
            return None
        elif spec.loader is None:
            namespace_path = spec.submodule_search_locations
            if namespace_path:
                # 这里会创建针对 Namespace package 的 Spec
                spec.origin = None
                spec.submodule_search_locations = _NamespacePath(fullname, namespace_path, cls._get_spec)
                return spec
            else:
                return None
        else:
            return spec
```

找到 Spec 以后，也就确定了要使用的 Loader。一般来说，Load 就是执行 `*.py` 文件的内容，将里面的属性绑定到 Module object 上面。最后再返回这个 Object 便完成了 Import。

# Partially Circular Import

对于下面这种情况：

```python
# foo.py
import bar

# bar.py
import foo
```

看起来会出现错误，其实是可以执行成功的。原因在于 Import 做了特殊处理，来看代码：

```python
def _load_unlocked(spec):
    # 这里的 initializing 对应了前面的 _lock_unlock_module 
    spec._initializing = True
    try:
        # 现在已经得到了 Spec，准备进行 Load
        # 但是在实际的 exec_module 之前会先将 Module object 放进 sys.modules
        # 这样在 Circular import 时缓存中就可以找到这个 Module object
        # 但是这个 Object 尚未完成加载，所以是 Partial 的
        sys.modules[spec.name] = module
        try:
            if spec.loader is None:
                if spec.submodule_search_locations is None:
                    raise ImportError('missing loader', name=spec.name)
            else:
                spec.loader.exec_module(module)
        except:
            try:
                del sys.modules[spec.name]
            except KeyError:
                pass
            raise
        module = sys.modules.pop(spec.name)
        sys.modules[spec.name] = module
        _verbose_message('import {!r} # {!r}', spec.name, spec.loader)
    finally:
        spec._initializing = False

    return module
```

当然，如果使用不当也是会引发异常的。

```python
# foo.py
from foo import a

a = 11

# 这种情况下 Import foo 会导致 ImportError
# ImportError: cannot import name 'a' from partially initialized module 'foo' (most likely due to a circular import)
```

```python
# foo.py
import bar

# bar.py
import foo

print(foo.a)

# 这种情况下 Import foo 会导致 AttributeError
# AttributeError: partially initialized module 'foo' has no attribute 'a' (most likely due to a circular import)
```

原因就是尝试在 `foo` 模块仅仅 Partially imported 的情况下获取尚不存在的属性。

# Reload

除了 `import_module`，`importlib` 还提供了一个 `reload` 接口，可以用来重新加载之前 Import 过的模块。

`reload` 会刷新 `sys.modules` 中的 Module object，这没有问题。但是，对于 `from foo import bar` 这样的 Import 方式，`bar` 是不会单独更新的。

```python
# foo.py
def bar():
    print('bar')
    
# 如果 import foo，然后这样使用 foo.bar()。Reload 是 OK 的。
# 如果再加上 from foo import bar，那么即使改变 bar 的代码再 reload(foo)，bar() 还是会保持原有的效果，即打印 `bar` 字符串。
```

从源码来理解，这是因为以上涉及到的 Import 实现都只针对 Module object 的创建和更新。从字节码可以看到：

```shell
➜ echo 'from os import path' | python -m dis
  1           0 LOAD_CONST               0 (0)
              2 LOAD_CONST               1 (('path',))
              4 IMPORT_NAME              0 (os)
              6 IMPORT_FROM              1 (path)
              8 STORE_NAME               1 (path)
             10 POP_TOP
             12 LOAD_CONST               2 (None)
             14 RETURN_VALUE
```

`path` 的 Name binding 是通过 `STORE_NAME` 指令完成的，而 Import 的代码只负责 `IMPORT_NAME` 和 `IMPORT_FROM`，所以 `reload` 才会有 Module level 的限制。

---

*References*

- [How the Python import system works](https://tenthousandmeters.com/blog/python-behind-the-scenes-11-how-the-python-import-system-works/)

- [Module attributes - Python Docs](https://docs.python.org/3/reference/import.html#import-related-module-attributes)
- [Reload - python3 Cookbook](https://python3-cookbook.readthedocs.io/zh_CN/latest/c10/p06_reloading_modules.html)
