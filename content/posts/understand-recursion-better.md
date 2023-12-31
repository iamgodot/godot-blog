---
title: "Understand Recursion Better"
date: 2021-11-08T18:18:10+08:00
categories:
  - Code
tags:
  - dsa
draft: true
---

在 [Simple Recursion](/posts/simple-recursion) 之后，我一度把递归当作一种算法。但通过比较 Divide and Conquer 和 Dynamic Programming，我才发现之前的理解有点问题。

一切还是要从 Algorithmic Paradigm 说起：

> An **algorithmic paradigm** or **algorithm design paradigm** is a generic model or framework which underlies the design of a class of [algorithms](https://en.wikipedia.org/wiki/Algorithm). An algorithmic [paradigm](https://en.wikipedia.org/wiki/Paradigm) is an abstraction higher than the notion of an algorithm, just as an algorithm is an abstraction higher than a [computer program](https://en.wikipedia.org/wiki/Computer_program).

算法范式，是在算法的层面上抽象出来的一种更泛化的思想，常用的有：

- Brute-force search 暴力解法
- Backtracking 回溯算法
- Greedy algorithm 贪心算法
- Divide and conquer 分治法
- Dynamic programming 动态规划

Divide and Conquer 的基本思路是把复杂的问题分解成多个类似的简单问题，解决之后再组合起来得到最终结果。这种算法有很多应用，比如排序中的 Merge Sort，先将数列分解成单个元素，然后再归并，这时子数组都已经排好顺序了，所以过程很快。

对于 DC 来说递归就成为一种很合适的实现方式，因为递归的调用是重复同一套逻辑，而 DC 中的问题和子问题也是相似的。但是这样性能会比较低下，因为很多子问题可能都是重复的，举 Fibonacci 的例子来说：

```python
def fib(n):
    if n <= 0:
        return 0
    if n == 1:
        return 1

    return fib(n-2) + fib(n-1)
```

由于重复的子问题太多，最后的时间复杂度会高达 O(2^n). 此时就需要利用 Memoization 来优化，也就是缓存中间的计算结果，比如下面这个版本：

```python
from functools import cache

@cache
def fib(n):
    if n <= 0:
        return 0
    if n == 1:
        return 1

    return fib(n-2) + fib(n-1)
```

而这种解法实际上可以归类到另一种范式：动态规划。

Dynamic Programming 也是把问题划分成一个个小的子问题来解决，它包含两个要素：

1. 问题的最优解可以看成子问题的最优解的组合
2. 子问题存在重复的情况，如 Fibonacci 的例子

如果不存在子问题的重复，而只需要求出各个子问题的最优解再合并的话，就变回 Divide and Conquer 而不算是 Dynamic Programming 了，比如 Merge Sort 和 Quick Sort.

动态规划一般有两种实现思路：top-down 和 bottom-up. 上面缓存的写法就是 top-down，相当于把 n 从大到小地解决。而 bottom-up 可以用迭代来实现：

```python
def fib(n):
    if n <= 0:
        return 0
    if n == 1:
        return 1

    x, y = 0, 1
    for _ in range(1, n):
        x, y = y, x + y

    return y
```

此时可以更好地看出来，递归和算法范式并不是同一层面的概念。

那么 Dynamic Programming 可以用递归实现 bottom-up 么？答案是肯定的，用尾递归：

```python
def fib(n, x=0, y=1):
    if n <= 0:
        return 0
    if n == 1:
        return 1
    if n == 2:
        return x + y

    return fib(n - 1, y, x + y)
```

为什么这样就是 bottom-up 了呢，注意 x 和 y 的变化，是从 0 和 1 开始变化的，累积到最后 x+y 作为最终结果，所以是自底向上地解决了问题。至于尾递归，知乎上[这个答案](https://www.zhihu.com/question/20761771/answer/57214778)解释得很好：

> 尾递归，比线性递归多一个参数，这个参数是上一次调用函数得到的结果；

> 所以，关键点在于，尾递归每次调用都在收集结果，避免了线性递归不收集结果只能依次展开消耗内存的坏处。

因为尾递归需要参数来保存中间结果，所以函数定义不像普通递归那么简洁，有两种方法可以解决，一种是像上面代码中一样使用参数默认值，还有一种就是[柯里化](https://en.wikipedia.org/wiki/Currying)，简单来说就是二次封装，在 Python 中可以用 partial 轻松完成:

```python
from functools import partial

f = partial(fib, x=0, y=1)
```

尾递归的效率很高，因为不需要返回上一级调用，所以不会出现栈溢出的情况。类似 GOTO 关键字，经过优化的汇编代码会直接 JUMP 到下一个调用而不用保留之前的内部信息。

但是这种优化需要编译器的支持，并不是所有语言都会兼容尾递归的优化，比如 Python 一直就没有 TCO(Tail Call Optimization). 其中一个重要原因就是会影响 Debug 时查看对应的栈信息，具体可以参考 Guido 叔的博客（见 References）。

同样，尾递归只是手段，可以用它来实现 Dynamic Programming，也可以用其他的方法，比如迭代（比如 Fibonacci 中迭代的复杂度更低，因为只需要 O(1) 的空间）。DP 的关键在于 Memoization 而不是递归，这也说明了方法和原理的关系，前者可以达到目的，但是代替不了后者本身。

---

*References*

- [Algorithmic paradigm - Wikipedia](https://en.wikipedia.org/wiki/Algorithmic_paradigm)
- [Divide and Conquer - Wikipedia](https://en.wikipedia.org/wiki/Divide-and-conquer_algorithm)
- [Dynamic Programming - Wikipedia](https://en.wikipedia.org/wiki/Dynamic_programming)
- [Tail Call - Wikipedia](https://en.wikipedia.org/wiki/Tail_call)
- [尾调用优化 - 阮一峰](https://www.ruanyifeng.com/blog/2015/04/tail-call.html)
- [柯里化 - Wikipedia](https://en.wikipedia.org/wiki/Currying)
- [Tail Recursion Elimination - Guido](http://neopythonic.blogspot.com/2009/04/tail-recursion-elimination.html)
- [Final Words on Tail Calls - Guido](http://neopythonic.blogspot.com/2009/04/final-words-on-tail-calls.html)
