---
title: "Look and Say"
date: 2021-11-07T16:59:50+08:00
categories:
  - Code
tags:
  - dsa
draft: true
---

无意中看到一种叫 look-and-say 的数列，很有意思，有点儿 Fibonacci 的感觉。数列如下：

> 1, 11, 21, 1211, 111221, 312211, 13112221, 1113213211, ...

从第二位开始，每个数字都是对前一个数的计数结果的描述。比如 11 表示前面的 1 有一个 1，而 21 表示前面的 11 有两个 1，1121 表示 21 有一个 2 和一个 1，依次类推，可以无穷循环出新的结果。（除了 22 这个数字，因为会一直重复 22 本身）

后来查了查才发现原来 Leetcode 也有这道题，不过名字叫做 [count-and-say](https://leetcode-cn.com/problems/count-and-say/)。基本就是给 n 然后求此数列的第 n 项结果。

想了想思路并不难，无非是对一个数字的所有位数循环再计数就好了，不过写的时候很犯难，竟然还写出了一个无比冗长的 for 循环加 if 面条代码。这让我意识到了自己对于循环的认识有多么不够深刻。

之所以犯难，其实是不知道怎样把几种逻辑合并在一起，如果粗暴地列举所有分支大概是这样：

```python
def find_next(num: str):
	result = ''
	cur, count = num[0], 1
    size = len(num)
    for i in range(1, size):
        if num[i] == cur and i == size - 1:
            # count 加一
            # 更新 result
        elif num[i] == cur:
            # count 加一
        elif i == size - 1:
            # 更新 result
            # 更新 cur&count
            # 更新 result
        else:
            # 更新 result
            # 更新 cur&count
    # 还有一种情况就是并没有进入 for 循环
    # 那么也需要更新 result
```

直接按照这些分支把代码填满肯定很难看，所以要挑选合并。看起来只有第二个分支不需要更新 result，于是我决定把它单拎出来：如果 `num[i]==cur` 那么 count 加一；否则就更新 result 以及 cur&count. 那么 `i==size-1` 的时候怎么办呢，既要保存当前结果又可能需要记录新的结果，一筹莫展。不过后来还是发现了巧妙的写法：

```python
def find_next(num: str):
    result = ''
    cur, count = num[0], 1
    for char in num[1:]:
        if char == cur:
            count += 1
            continue
        result += f'{count}{cur}'
        cur = char
        count = 1

    return f'{result}{count}{cur}'
```

优点在于结合了 count&cur 放在返回值里面，通过这两个变量做缓存，就不需要单独考虑循环到最后一位和没有进入循环的情况了，于是 for 循环也可以直接遍历元素值而不是下标。

我正觉得这段写得精彩，却又看到了使用 while 的解法。什么，为什么会是 while 呢，明明容器长度是已知的，要用 for 才对啊。先看代码：

```python
def find_next(num: str):
    result = ''
    start = cur = 0
    size = len(num)
    while cur < size:
        while cur < size and num[cur] == num[start]:
            cur += 1
        result += str(cur - start) + num[start]
        start = cur

    return result
```

虽然我觉得 while 循环是比 for 的可读性要差的，但不得不承认这里的嵌套 while 写法更好，关键在于更符合思考的过程。如果是手动来数的话，也是整体一个大循环，然后对重复的数字不断加一，直到有变化，就先记录前面的结果再继续数新的数字。反而在上面巧妙的 for 循环中，是不太容易想到返回时把 result, count, cur 三者组合到一起的。

而且这里的 while 代码也很简洁。既复用了内部循环的 `cur += 1`，又节省了单独的 count 变量来计数。

所以大概可能也许是，虽然已知元素的总数，但是在更符合抽象逻辑的情况下用 while 会更好吧。

说来自己写循环的时候总会遇到几个惯性 bug：

- 下标越界或者下标类型不是 int
- 遍历的过程中修改更新容器本身
- 写 while 不注意退出条件导致无限循环

不知道有没有更智能的编辑器（插件）能自动识别循环代码中的 bug 呢 - -.

---

*References*

- [Look and say - Wikipedia](https://en.wikipedia.org/wiki/Look-and-say_sequence)
- [Count and say - Leetcode](https://leetcode-cn.com/problems/count-and-say/)
