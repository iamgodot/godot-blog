---
title: "Sourcecode of Python Heap Queue"
date: 2021-11-29T23:26:37+08:00
categories:
  - Code
tags:
  - sourcecode
  - python
  - dsa
draft: false
---

Heap 作为一种重要的数据结构，有许多应用场景，比如优先级队列，每次出队的都是最大值或者最小值的元素。很多语言都集成了相关实现，比如 Java 的 PriorityQueue，而 Python 提供了 heapq 模块。

因为 Heap 通常用数组而不是链表存储，所以 Python 里面的 Heap 实质上就是一个列表，而 heapq 提供的几个函数也是以列表对象作为参数的：

```python
from heapq import heappush, heappop, heappify, heapreplace, heappushpop

heap = []
heappush(heap, 1)
item = heap[0]  # 第一个元素代表堆顶元素
heappop(heap)
heapify([3, 2, 1, 5, 6, 4])  # 把普通列表转化为堆结构 [1, 2, 3, 4, 5, 6]
heapreplace([3, 4, 5], 1)  # 直接将堆顶元素 3 替换为 1，最后堆结构为 [1, 4, 5]
heappushpop([3, 4, 5], 1)  # 先将 1 插入堆中，再 pop 出堆顶元素，最后堆结构为 [3, 4, 5]
```

为什么 heapq 提供的是最小堆而不是更常见的最大堆呢？这就得从源码中找答案了。

> Our API differs from textbook heap algorithms as follows:
>
> - We use 0-based indexing.  This makes the relationship between the
>   index for a node and the indexes for its children slightly less
>   obvious, but is more suitable since Python uses 0-based indexing.
>
> - Our heappop() method returns the smallest item, not the largest.
>
> These two make it possible to view the heap as a regular Python list
> without surprises: heap[0] is the smallest item, and heap.sort()
> maintains the heap invariant!

也就是说，为了保证第一个元素是最小值，以及列表的 sort 方法不会破坏堆结构（注意这里的结构不是说元素的绝对顺序，而是 heap[k] <= heap[2k+1] & heap[k] <= heap[2k+2] 这两个不等式），所以提供的是最小堆的接口。实际上，其实最大堆的逻辑也都已经实现了，只是作为内部函数没有暴露出来。

Heap 本身并不复杂，但从 heapq 的代码中可以感受到，在基础概念之上，作者还结合了实际的应用场景来选择最佳的实现方案。

# Sift Up

每次的 push 和 pop 操作，都要调整堆结构，因此代码中封装了 `_siftdown` 和 `_siftup` 两个函数，前者是在插入叶子节点后自底向上进行更新，而后者则是把叶子节点放到根节点的位置再从上到下做调整。特别之处是，对于 `_siftup`，作者没有从根节点开始向下依次做比较，而是先把根节点迅速交换到最下面，然后直接调用 `_siftdown`:

```python
def _siftup(heap, pos):
    endpos = len(heap)
    startpos = pos
    newitem = heap[pos]
    # Bubble up the smaller child until hitting a leaf.
    childpos = 2*pos + 1    # leftmost child position
    while childpos < endpos:
        # Set childpos to index of smaller child.
        rightpos = childpos + 1
        if rightpos < endpos and not heap[childpos] < heap[rightpos]:
            childpos = rightpos
        # Move the smaller child up.
        heap[pos] = heap[childpos]
        pos = childpos
        childpos = 2*pos + 1
    # The leaf at pos is empty now.  Put newitem there, and bubble it up
    # to its final resting place (by sifting its parents down).
    heap[pos] = newitem
    _siftdown(heap, startpos, pos)
```

原因是在实际情况中，叶子节点的值通常都很大，所以如果从根节点的位置向下比较，会迭代很多次。而切换到最下层之后，上浮调整的次数会明显小很多。从作者提供的数据来看，这种改动让 `heappop` 操作的 comparisons 近乎减半：

```python
# On random arrays of length 1000, making this change cut the number of
# comparisons made by heapify() a little, and those made by exhaustive
# heappop() a lot, in accord with theory.  Here are typical results from 3
# runs (3 just to demonstrate how small the variance is):
#
# Compares needed by heapify     Compares needed by 1000 heappops
# --------------------------     --------------------------------
# 1837 cut to 1663               14996 cut to 8680
# 1855 cut to 1659               14966 cut to 8678
# 1847 cut to 1660               15024 cut to 8703
```



# Merge

除了基础操作，heapq 还基于 Heap 封装了几个 higher level 的函数：

- `merge`
- `nsmallest`
- `nlargest`

如果你需要归并若干个已经有序的列表又不想一次性把所有数据都读取到内存的话，那么 merge 是一个好选择。它的实现巧妙地利用了 Heap:

```python
def merge(*iterables, key=None, reverse=False):
    h = []
    h_append = h.append

    if reverse:
        _heapify = _heapify_max
        _heappop = _heappop_max
        _heapreplace = _heapreplace_max
        direction = -1
    else:
        _heapify = heapify
        _heappop = heappop
        _heapreplace = heapreplace
        direction = 1

    if key is None:
        for order, it in enumerate(map(iter, iterables)):
            try:
                next = it.__next__
                h_append([next(), order * direction, next])
            except StopIteration:
                pass
        _heapify(h)
        while len(h) > 1:
            try:
                while True:
                    value, order, next = s = h[0]
                    yield value
                    s[0] = next()           # raises StopIteration when exhausted
                    _heapreplace(h, s)      # restore heap condition
            except StopIteration:
                _heappop(h)                 # remove empty iterator
        if h:
            # fast case when only a single iterator remains
            value, order, next = h[0]
            yield value
            yield from next.__self__
        return
    
    ...
```

最后面省略的是当 key 不为 None 时的处理，思路和前面基本一致。

首先从对 reverse 的判断部分可以看出，heapq 是实现了最大堆的接口的。

关键部分在于下面的循环中对堆结构的使用，注意到存入 Heap 的数据并不是各个列表的元素，而是由头元素、序号和迭代器三部分组成（针对每个列表）。在 heapify 时比较的就是各个列表的头元素大小，如果相同，那么就看列表们在传参进来的顺序。这样就做到了稳定排序，一个是列表之间的等值元素会按照列表的顺序排列，而某个列表中的重复元素也不会打乱原有顺序。在 while 循环中，每输出一个元素，都会重新调整 Heap 结构，以此来选择下一个列表。

如果总共只传入了一个列表，那么就直接短路 while 的部分，直接使用列表的迭代器。这种优化在 heapq 中有很多，通过区分边界情况使用不同的方案，达到整体复杂度的最优。

# Find First N

最后看下 `nsmallest` 函数，它可以用来返回一个集合中最小的 n 个元素。类似 Leetcode 上面 [Kth Largest Element](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/) 那道题，heapq 选择了维护一个大小为 k 的 Heap:

```python
def nsmallest(n, iterable, key=None):
    ...
    
    # When key is none, use simpler decoration
    if key is None:
        it = iter(iterable)
        # put the range(n) first so that zip() doesn't
        # consume one too many elements from the iterator
        result = [(elem, i) for i, elem in zip(range(n), it)]
        if not result:
            return result
        _heapify_max(result)
        top = result[0][0]
        order = n
        _heapreplace = _heapreplace_max
        for elem in it:
            if elem < top:
                _heapreplace(result, (elem, order))
                top, _order = result[0]
                order += 1
        result.sort()
        return [elem for (elem, order) in result]
    
    ...

```

这里仍然用到了 order 作为序号的方式来保证稳定。另外，获取最终的 result 后，直接应用 sort 方法而不是继续用 Heap 输出，这样写很简便，而且堆排序也不会比 [Timsort](https://en.wikipedia.org/wiki/Timsort) 更优。

理论上 `nsmallest` 和 `sorted(iterable, key=key)[:n]` 的效果是一样的，但是前者的可读性更好，而后者则不需要引入额外的模块。

在函数的注解中，作者详细列出当前实现的比较次数和复杂度，同时对比了其他的方案，非常值得一读。

---

*References*

- [Heap - Wikipedia](https://en.wikipedia.org/wiki/Heap_(data_structure))
- [Timsort - Wikipedia](https://en.wikipedia.org/wiki/Timsort)
- [COMPARE ALGORITHMS FOR HEAPQ.SMALLEST](https://code.activestate.com/recipes/577573-compare-algorithms-for-heapqsmallest/)
- [Kth Largest Element](https://leetcode-cn.com/problems/kth-largest-element-in-an-array/)
