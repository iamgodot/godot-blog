---
title: "Understand Recursion Better"
date: 2021-11-08T18:18:10+08:00
categories:
  - Code
draft: false
---

在 [Simple Recursion](/posts/simple-recursion.md) 之后，我一度把递归当作一种算法来学习。直到把它与 Divide and Conquer 和 Dynamic Programming 放在一起理解，我才明白，递归只是实现方法，但它和后两者之间又存在千丝万缕的联系。

一切还是要从 Algorithmic Paradigm 说起：

> An **algorithmic paradigm** or **algorithm design paradigm** is a generic model or framework which underlies the design of a class of [algorithms](https://en.wikipedia.org/wiki/Algorithm). An algorithmic [paradigm](https://en.wikipedia.org/wiki/Paradigm) is an abstraction higher than the notion of an algorithm, just as an algorithm is an abstraction higher than a [computer program](https://en.wikipedia.org/wiki/Computer_program).

这就是算法范式，也就是说在算法的层面上抽象出来的一种更泛化的思想，比如常用的有：

- Brute-force search 暴力解法
- Backtracking 回溯算法
- Greedy algorithm 贪心算法

还有两种就是一开始提到的分治法与动态规划。

Divide and Conquer 的基本思路是把复杂的问题分解成多个类似的简单问题，解决之后再组合起来得到最终结果。这种算法有很多应用，比如排序中的 Merge Sort，先将数列分解成单个元素，然后再分别聚合，由于聚合的过程都是已经排好顺序的数组，所以效率会高很多。

对于 DC 来说递归就成为一种很合适的实现方式，因为递归的调用是重复同一套代码逻辑，而 DC 中的子问题和最终问题也都是相似的。

然而 DC 有个问题是可能会不断地碰到相同的子问题，如果每次都从头解决一遍会造成很严重的性能问题，拿 Fibonacci 举例：

```python
def fib(n):
    if n <= 0:
        return 0
    if n == 1:
        return 1

    return fib(n-2) + fib(n-1)
```

由于重复的子问题太多，这么写最后会得到 O(2^n) 的时间复杂度。

对于这种情况，就需要使用 Memoization，即缓存中间的计算结果，于是得到下面的版本：

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

而这种解法的思路其实就来自另一种算法：动态规划。

Dynamic Programming 也是把问题划分成一个个小的子问题来解决，它包含两个要素：

1. 问题的最优解可以看成子问题的最优解的组合
2. 子问题存在重复的情况，如 Fibonacci 的例子

如果不存在子问题的重复，而只需要求出各个子问题的最优解再合并的话，就变回 Divide and Conquer 而不算是 Dynamic Programming 了，比如 Merge Sort 和 Quick Sort.

动态规划一般有两种实现思路：top-down 和 bottom-up. 上面利用缓存的写法就是 top-down，相当于把 n 从大到小地解决。而 bottom-up 可以用迭代来实现：

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

到此基本可以看出来，递归是代码实现的方法，和两种算法并不是同一层面的概念。

那么 Dynamic Programming 可以用递归实现 bottom-up 么？答案是肯定的，用尾递归。

像下面这么写：

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

为什么这样就是 bottom-up 了呢，注意 x 和 y 的变化，是从 0 和 1 开始变化，也就是自底向上地解决了问题。至于尾递归，知乎上[这个答案](https://www.zhihu.com/question/20761771/answer/57214778)解释得很好：

> 尾递归，比线性递归多一个参数，这个参数是上一次调用函数得到的结果；
> 所以，关键点在于，尾递归每次调用都在收集结果，避免了线性递归不收集结果只能依次展开消耗内存的坏处。

因为尾递归需要参数来保存中间结果，所以函数定义就多了参数，调用起来不像普通递归那么简洁，有两种方法可以应对，一种是像上面代码中借用参数默认值来规避，还有一种就是[柯里化](https://en.wikipedia.org/wiki/Currying)，简单说就是二次封装，在 Python 中用 partial 可以轻松完成:

```python
from functools import partial

f = partial(fib, x=0, y=1)
```

尾递归的效率很高，因为不需要返回上一级调用，所以不存在栈溢出的隐患。类似 GOTO 关键字，经过编译器优化的汇编代码会直接 JUMP 到下一个调用而不用保留之前的内部信息。

但是对于 Python 来说并没有用，因为不支持 TCO(Tail Call Optimization). 从 Guido 叔的解释看，有一个重要原因是会影响 Debug 时查看对应的栈信息，同时他还给出了一种简化 TCO 实现的写法：与其用一堆 If 分支跳来跳去，可以不这么写 `return foo(args)` 而写成 `return foo, (args, )`.

同样，尾递归只是一种实现的手段，可以用它来完成 Dynamic Programming，也可以用其他的方法，比如迭代（对于 Fibonacci 的例子来说迭代的复杂度更低，因为只需要 O(1) 的空间），因为 Dynamic Programming 的关键在于 Memoization.

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
