---
title: "关于 Pager"
date: 2022-04-02T14:09:53+08:00
categories:
  - Code
tags:
  - python
draft: false
keywords:
  - Python pager
  - Python help pager
  - Pydoc pager
---

好久没有更新了，最近研究了下如何用 Python 实现 Pager 的功能，这里指的是 Terminal 中的 Paging 程序，比如 `less`。

# Why Pager

Pager 在大段文字的展示中很常见，比如 Linux 的 `man page`，而 `$PAGER` 就是用来指定默认 Paging 程序的环境变量。Python shell 里面的 `help()` 会默认翻页显示，IPython 的 `?` 则更胜一筹，能够判断当前屏幕的可用空间来决定是否 Paging。

`Less` 应该算是最流行的 Pager 了，相比于 `more`，它同时支持前进和后退翻页，而且因为不需要一次性读取整个文件，它的启动速度在打开大型文件时要远远快于 `vi`。因此，许多 Pager 都是通过启动系统自带的 `less` 程序来实现。

# Don't Reinvent the Wheel

轮子总是有的，而且还很多，这里说几个比较好用的：

- `Pydoc` 的 `pager`
- `Click` 的 `echo_via_pager`
- `IPython` 的 `page`

`Pydoc` 是 Python 自带的，已经稳定存在了很多年，轻巧好用；`Click` 的实现类似，而且支持传入一个 `generator`；`IPython` 的 `page` 更加强大，可以自动判断当前的屏幕大小，再结合一个 `screen_lines` 参数来计算最终的可用空间。

再说说这几个轮子的实现，基本思路都是上面提到的调用系统 Pager。因为要兼容五花八门的操作系统，大致上又分为三种处理方式：

- 理想情况下是使用 `PIPE`。因为打开的系统 Pager 必然是子进程，而 `PIPE` 通过内存中的缓冲区实现了 `IPC`，这样既不用一次性读取所有数据，后续的 `write` 操作效率也高。

- 如果 `PIPE` 不可用，那就需要先建立临时文件，把数据写入，最后再启动 Pager 直接打开文件。

- 保底的方案是直接向标准输出写入文本数据，这种做法一般只会出现在非 `tty` 的设备上。

另外 `IPython` 内部使用了 `curses` 库来检测屏幕大小，如果传入的 `screen_lines` 小于等于 0，就会加上检测结果的值作为最终的可用行数。从[官方文档](https://docs.python.org/3/howto/curses.html)上看 Windows 的 Python 没有 `curses`，但提供了 ported version 的 `UniCurses`，不知道 `IPython` 的屏幕检测在 Windows 的表现如何。

`Click` 虽然没有直接使用 `Pydoc`，但实现基本和后者相同；而 `IPython` 的轮子是从头造起，质量很高，值得一看。

# Do it Yourself

抛开那些杂乱的系统判断的部分，直接实现一个调用 `less` 的 Pager 还是很简单的。大致代码如下：

```python
from subprocess import PIPE, Popen

def pager(text: str):
    proc = Popen(
        'less',
        shell=True,
        stdin=PIPE,
        text=True,
    )
	proc.communicate(text)
```

这样会一次性把文本写入到 `PIPE` 中，对于一个生成器来说，不断迭代出新内容并写入会更高效：

```python
from subprocess import PIPE, Popen

def pager(generator):
    proc = Popen(
        'less',
        shell=True,
        stdin=PIPE,
        text=True,
    )
    try:
        for text in generator:
            proc.stdin.write(text)
    except (OSError, KeyboardInterrupt):
        pass
    else:
        proc.stdin.close()
    proc.wait()
```

虽然代码不麻烦，但在使用 `subprocess` 这个库的过程中踩了许多坑，需要对着文档来回确认。比如上面的 `Popen` 中的 `text` 参数是为了控制文本模式打开文件，但如果不传而是直接指定 `encoding` 或 `errors` 的话也会默认使用文本打开。。

下面再啰嗦几句 `subprocess`，如果没兴趣可以跳过：

- 现在的 Higher API 只关注 `run()` 就好了，以前的 `call/check_call/check_output` 都可以忽略，`run()` 中都提供对应参数
- 需要和子进程交互的话用 `Popen()`
- 对于不指定 `shell` 的 `Popen()`
  - 字符串作为命令：命令中无法携带参数，只可以这样 `Popen('echo', ...)`
  - 列表作为命令：所有元素会合在一起，所以可以 `Popen(['echo', 'foo', 'bar'])`
  - 如果命令出错或不存在会直接触发 `FileNotFound` 异常
- 对于 `shell=True` 的 `Popen()`
  - 官方文档直接推荐使用字符串，原因见下一行
  - 如果使用列表的话，只有第一个元素作为命令内容，其他的都会读取为 shell 的参数
  - 不会引发 `FileNotFound` 异常而是输出到 `stderr`

槽点满满，一开始用起来痛不欲生，不过也可能是这个库的历史包袱太重了吧。

---

小小的 Pager 并不难，但兼容各种 `OS` 和 `Terminal` 真的也需要勇气，更不用说有时候 `I/O` 操作也表现不一。生活不易，对于这种底层功能，真的不如拿来主义呀。
