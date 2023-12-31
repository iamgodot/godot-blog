---
title: "(Re)Write recursions easily"
date: 2021-10-25T08:56:54+08:00
categories:
  - Code
tags:
  - dsa
draft: true
---

近来刷题，觉得递归实在神奇。对于有的题目，用循环实现思路很清晰，改成递归却变得非常抽象；另一些相反，递归的做法既简洁又容易理解，但换成迭代就怎样都不明白了。经过几次痛苦的思考之后，我发现其中还是有迹可循的，按照套路来做，至少思路不会走得太偏。

在维基上面发现的关于递归的笑话：

> To understand recursion, you must understand recursion.

# Write Recursion

审题分析过后，如果选择用递归，需要先明确两个最重要的组成部分：

1. 边界条件，保证函数能够返回，不会无限递归下去
2. 递进关系，也就是上下层递归调用结果的等式规律

比如求 factorial：

```python
def factorial(n):
    if n <= 2:
        return n
    return factorial(n - 1) * n
```

如果必要的话，还得考虑的是如何维护上下文的状态信息，目的是计算最终的返回值，有两种办法：

1. 使用全局（outer scope）变量
2. 利用参数进行传递

下面的写法就是利用了参数来保存中间的计算结果：

```python
def factorial(n, res):
    if n <= 2:
        return res * n
    return factorial(n - 1, res * n)
```

也就是所谓的尾递归，对于可以优化尾递归的语言确实可以看成这种方式是从上至下的，而常规的递归则是从下到上。但我理解递归都是先下去再上来的，尾递归只是选择了一去不返而已。

上面的例子比较简单，反转链表的代码就比较难理解一点：

```python
def reverse_list(head):
    if not head or not head.next:
        return head
    new_head = reverse_list_by_recursion(head.next)
    head.next.next = head
    head.next = None
    return new_head
```

其实精髓在于倒数第二三行，因为抛开边界条件和递归调用之外，还需要增加每个节点掉转的辅助逻辑。

然而有时递归并不是重点，如果没有找好规律仍然是写不出来的，比如二叉树的层序遍历：

```python
def levelorder(root):
    res = []
    def _order(root, level):
        if not root:
            return
        if len(res) == level:
            res.append([])
        res[level].append(root.val)
        _order(root.left, level + 1)
        _order(root.right, level + 1)

    _order(root, 0)
    return res
```

这里的上下文既用到了外层变量，也借助了中间参数，但难点并不在这里，而是在递归的思想下实现 BFS. 举这个例子想说的是递归题写不出来，不一定是递归学的不好，可能只是别的规律没有摸清楚。

写完之后，做一些检查也是必要的：

1. 极端参数值的情况是否满足要求，比如 n < 0

2. 是否能够保证正常跳出，是否会导致栈溢出

3. 递归深度是否过多，数据有没有可能溢出

4. 是否存在重复计算，见下面的 Fibonacci

5. 是否可以做尾递归优化，或者用循环实现

# Rewrite Recursion

对递归的模拟当然离不开循环，但是有的时候未必需要用栈，还有时候则非要用不可。

第一种针对的递归通常比较简单，比如经典的 Fibonacci:

```python
from functools import cache

@cache
def fib(n):
    if n <= 1:
        return n
    else:
        return fib(n-2) + fib(n-1)
```

因为这种递归时间复杂度会高达 2^n，所以要加上 cache 保存数据。那么如何改写呢，非常简单，甚至不需要基于递归的写法思考：

```python
def fib(n):
    if n <= 1:
        return n

    x, y = 0, 1
    for _ in range(1, n):
        x, y = y, x + y

    return y
```

另外像求 power 或者 factorial 都是类似的情况。

但第二种就会比较复杂，也就是需要使用 stack 这种数据结构，实际上编译器在背后也是以这样的方式实现递归代码的。举个快速排序的例子来说明，首先是递归做法：

```python
def quick_sort(nums, left, right):
    if left >= right:
        return

    pivot, i, j = left, left, right
    while i < j:
        while nums[j] >= nums[pivot] and i < j:
            j -= 1
        while nums[i] <= nums[pivot] and i < j:
            i += 1
        if i < j:
            nums[i], nums[j] = nums[j], nums[i]

    nums[pivot], nums[i] = nums[i], nums[pivot]
    quick_sort(nums, left, i - 1)
    quick_sort(nums, i + 1, right)
```

前面的逻辑都在把数列分成大小两部分，重点在最后两行，也就是按照 Divide and Conquer 的思路对左右两组数分别继续进行排序。

那么如何改写呢，先继续递归的逻辑往后推想，如果开始给左边组数排序的话，肯定要保留现场，这里就是右边组数的 index，即 i + 1 和 right 的值，不然回来的时候就定位不到了。这个保留操作就对应入栈操作。

同理，何时出栈呢，肯定是递归调用返回的时候，要用保存的 index 做情景再现。

于是有了迭代的写法：

```python
def quick_sort_with_stack(nums, left, right):
    stack = list()
    while left < right or stack:
        if left < right:
            pivot, i, j = left, left, right
            while i < j:
                while nums[j] >= nums[pivot] and i < j:
                    j -= 1
                while nums[i] <= nums[pivot] and i < j:
                    i += 1
                if i < j:
                    nums[i], nums[j] = nums[j], nums[i]

            nums[pivot], nums[i] = nums[i], nums[pivot]
            stack.append([i + 1, right])
            right = i - 1
        else:
            left, right = stack.pop()
```

可以看到，对应递归写法中每次进入下一层的时候，这里就会做入栈保存，然后更新变量（right），来模拟新的调用过程；而返回的时候，又会出栈，复原原来的 left 和 right.

还有就是外层的 while 循环，前面的条件对应快排逻辑本身的要求，后者控制递归调用返回的时机。

另外一个经典题目是二叉树的中序遍历，递归版本非常简单：

```python
def inorder(root, result):
    if not root:
        return
    inorder(root.left)
    result.append(root.val)
    inorder(root.right)
```

可以看到和快速排序的区别是，需要先处理左子节点，然后是将节点值加入列表的逻辑，最后再处理右子节点。因为上来就做递归调用，所以改写的话也要先进行入栈操作，直到开始出栈的时候再执行逻辑，之后再（为右子节点）入栈出栈。实现如下：

```python
def inorder(root):
    stack, res = [], []
    cur = root
    while cur or stack:
        if cur:
            stack.append(cur)
            cur = cur.left
        else:
            cur = stack.pop()
            res.append(cur.val)
            cur = cur.right

    return res
```

其实 while loop 和 stack push/pop 都和快排差不多，只是对于栈来说，快排会先全部入栈再出栈，而这里会入栈出栈多遍，分别对应左子节点们和右子节点们。原因就是因为 append 逻辑夹在了两个递归调用之间，如果是先序遍历则会简单很多。

总结起来就是确定要不要用栈，如果需要的话，想清楚：

1. while 循环的条件，针对 base case 和 stack
2. 栈保存的数据是什么，用什么结构
3. 入栈和出栈的时间点

---

递归是一种很聪明的做法，逻辑简单清晰，可以把代码写得很优雅。缺点也很明显，极端情况下容易导致栈溢出，而且如果没有优化好，复杂度也可能会很高，毕竟用（普通）递归首先就难免调用带来的栈空间消耗。

又想到维基上还有另一个笑话：

> Recursion, see Recursion.

---

*References*

- [Recursion - Wikipedia](https://en.wikipedia.org/wiki/Recursion_(computer_science))
- [如何写递归 - Leetcode](https://leetcode-cn.com/circle/article/koSrVI/)
