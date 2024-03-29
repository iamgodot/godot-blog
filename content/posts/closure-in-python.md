---
title: "Python 中的闭包"
date: 2022-01-29T17:55:18+08:00
categories:
  - Code
tags:
  - python
draft: true
keywords:
  - Python 闭包
  - Python 的 Closure 是什么
  - Python Closure
---

```python
def f1():
    l = []
    def c():
        l.append(1)

def f2():
    a = 1
    def c():
        print(a)
        a = 2
```

类似 f1 和 f2 中的闭包写法，之前总是在用了前者多次之后，再写后一种就报错，感觉很莫名其妙，明明都差不多。研究了下发现，其实这是 Python 闭包的 ~~Bug~~Feature，理解之后反而觉得这样的设计是合理的。

首先说 Closure，也就是闭包，特指内部的函数及其引用的超出本身作用域的对象，总是在函数嵌套时发生。在 f1 中，c 就是一个闭包函数，同时 l 也算作其中的一部分。因为 c 使用了 l 导致延伸了原有的作用域，所以才有闭包的产生。

再看 f2，如果我们只对 a 做 read 操作是不会引发问题的，由于 c 中尝试了赋值操作，才导致了 `UnboundLocalError`. 这是因为 Python 解释器会假定函数中赋值的变量是局部变量，而 c 中本身并没有定义 local 的 a 变量；其次，异常在 `print(a)` 时就会抛出，不会等到 `a = 2` 的执行。

那为什么 f1 没问题呢，是因为列表为可变对象，append 操作只是更新了里面的内容，并不存在赋值。

```python
def f():
    l = []
    def c():
        l.append(1)
    return c
```

像上面这样，我们可以通过执行 f 得到闭包函数 c. 此时 f 已经执行完毕，它的作用域也随之消失，l 则作为自由变量绑定到了 c 上。和 c 关联的所有自由变量名称都可以在 `c.__code__.co_freevars` 中看到，而真正的对象则保存在 `c.__closure__` 里面。

怎样才能给这些自由变量赋值呢？Python 提供了 `nonlocal` 关键字，顾名思义，它表示一个变量来自外层函数，此时再进行赋值就不会出现前面的问题了：

```python
def f():
    a = 1
    def c():
        nonlocal a
        print(a)
        a = 2
```

其实除了 `nonlocal` 之外还有个 `global` 关键字，目的是指定使用 Global 命名空间的变量，也就是一个模块（文件）的作用域下。

作用域可以简单理解成变量的作用范围，`a = []` 这样的赋值语句把变量 a 和列表对象做了绑定，之后 a 这个名称可以使用的范围就是它的作用域。而同一层作用域的所有绑定了的变量名和对象都存放在一个数据结构内，即命名空间，Python 的命名空间就是 Dict 来实现的。

包括 Global，Python 一共有 LEGB 四大命名空间，一圈圈扩大，但是不相互覆盖：

- Local: 函数内部
- Enclosing: 闭包内部
- Global: 模块内部
- Builtin: 内建的函数和其他对象

还有一点要注意的是，和函数不同，在类的内部是无法使用闭包的，必须要以 `self.a` 或者 `cls.a` 的方式来引用类变量。

---

*References*

- Fluent Python
