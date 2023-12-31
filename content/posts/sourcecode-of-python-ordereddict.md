---
title: "Python OrderedDict 实现 LRU 缓存"
date: 2021-11-28T17:50:51+08:00
categories:
  - Code
tags:
  - python
  - dsa
draft: true
keywords:
  - Python OrderedDict 用法
  - Python OrderedDict 实现 LRU
  - Python 实现 LRU 缓存
---

LRUCache 是一种经典的缓存机制，它的基本思路是按照最近使用的时间对元素排序，在清理时优先把搁置最久的删除掉。

如果不想给每个缓存元素都记录一个时间戳的话，可以应用哈希链表来简单地实现 LRU 算法。也就是对一个哈希表中的所有元素增加指针，从而串起一个双链表，这样既可以快速 get value，又可以通过把最近使用过的元素放到头部来维护顺序，删除的时候从末尾开始就好了。

手写双链表并不困难，但是借助 OrderedDict 的话，可以写出非常简短的代码：

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.hashtable = OrderedDict()

    def get(self, key: int) -> int:
        if key in self.hashtable:
            self.hashtable.move_to_end(key, last=False)
            return self.hashtable[key]

        return -1

    def put(self, key: int, value: int) -> None:
        self.hashtable[key] = value
        self.hashtable.move_to_end(key, last=False)
        if len(self.hashtable) > self.capacity:
            self.hashtable.popitem()
```

其中最神奇的就是 `move_to_end` 和 `popitem` 方法（后者默认是弹出最后面的元素）的使用，这也得益于 OD 可以保证 key-value pair 的顺序。那么

OD 是如何做到的呢？其实还是双链表，下面是它的 Python 实现：

```python
class _Link(object):
    __slots__ = 'prev', 'next', 'key', '__weakref__'

class OrderedDict(dict):
    def __init__(self, other=(), /, **kwds):
        '''Initialize an ordered dictionary.  The signature is the same as
        regular dictionaries.  Keyword argument order is preserved.
        '''
        try:
            self.__root
        except AttributeError:
            self.__hardroot = _Link()
            self.__root = root = _proxy(self.__hardroot)
            root.prev = root.next = root
            self.__map = {}
        self.__update(other, **kwds)

    def __setitem__(self, key, value,
                    dict_setitem=dict.__setitem__, proxy=_proxy, Link=_Link):
        'od.__setitem__(i, y) <==> od[i]=y'
        # Setting a new item creates a new link at the end of the linked list,
        # and the inherited dictionary is updated with the new key/value pair.
        if key not in self:
            self.__map[key] = link = Link()
            root = self.__root
            last = root.prev
            link.prev, link.next, link.key = last, root, key
            last.next = link
            root.prev = proxy(link)  # 为了保证不出现循环引用
        dict_setitem(self, key, value)

    def __delitem__(self, key, dict_delitem=dict.__delitem__):
        'od.__delitem__(y) <==> del od[y]'
        # Deleting an existing item uses self.__map to find the link which gets
        # removed by updating the links in the predecessor and successor nodes.
        dict_delitem(self, key)
        link = self.__map.pop(key)
        link_prev = link.prev
        link_next = link.next
        link_prev.next = link_next
        link_next.prev = link_prev
        link.prev = None
        link.next = None

    def popitem(self, last=True):
        '''Remove and return a (key, value) pair from the dictionary.

        Pairs are returned in LIFO order if last is true or FIFO order if false.
        '''
        if not self:
            raise KeyError('dictionary is empty')
        root = self.__root
        if last:
            link = root.prev
            link_prev = link.prev
            link_prev.next = root
            root.prev = link_prev
        else:
            link = root.next
            link_next = link.next
            root.next = link_next
            link_next.prev = root
        key = link.key
        del self.__map[key]  # 这里并没有手动解除指针，所以有可能造成循环引用（prev 不使用 weakref 的话）
        value = dict.pop(self, key)
        return key, value

    def move_to_end(self, key, last=True):
        '''Move an existing element to the end (or beginning if last is false).

        Raise KeyError if the element does not exist.
        '''
        link = self.__map[key]
        link_prev = link.prev
        link_next = link.next
        soft_link = link_next.prev
        link_prev.next = link_next
        link_next.prev = link_prev
        root = self.__root
        if last:
            last = root.prev
            link.prev = last
            link.next = root
            root.prev = soft_link
            last.next = link
        else:
            first = root.next
            link.prev = root
            link.next = first
            first.prev = soft_link
            root.next = link
```

源码还有很多其他方法，这里只展示了关键的几个。可以看到，重要的数据结构在于 `self.__root` 为首的双链表以及 `self.__map` 来保存 key-link 的映射关系。这其实和哈希链表实现 LRUCache 的思路如出一辙：双链表维护顺序，字典（哈希表）快速定位元素。

`Link` 的定义很精简，利用了 `__slots__` 来节省空间。

`self.__root` 作为伪头部节点，这样整个双链表的操作会很统一（不过感觉 root 节点不用 weakref 也没有什么问题）。

值得一提的是 `weakref` 的部分，注意在 `__setitem__` 方法中，root 的前置节点使用了 `weakref.proxy` 来定义，这是为了避免在 `popitem` 的时候出现虽然删除了 `self.__map` 中的 link 但仍然释放不了内存的情况。

另外在 `__delitem__` 最后两行，代码显式地把 link 的指针置为 None，其实不这么写的话当方法结束时 link 被回收之后也是一样的，不过这样更：

> Explicit is better than implicit.

OD 和普通的 Python dict 在比较上也不一样：

```python
    def __eq__(self, other):
        '''od.__eq__(y) <==> od==y.  Comparison to another OD is order-sensitive
        while comparison to a regular mapping is order-insensitive.

        '''
        if isinstance(other, OrderedDict):
            return dict.__eq__(self, other) and all(map(_eq, self, other))
        return dict.__eq__(self, other)
```

两个 OD 对象比较时会严格按照元素的顺序，而 OD 和 dict 比较则又会忽略顺序了。

OD 拥有 `__dict__` （dict 没有），所以可以使用 `d.foo = bar` 这种属性赋值操作。

自 Python3.6 开始，普通 dict 也默认会按照插入的顺序保存元素，不过如果想显式地表达 order matters，还是 OD 更合适，而且它提供的 `move_to_end` 和 `popitem` 方法也更方便。相比之下，由于 dict 高效的实现，OD 的操作性能和内存占用都要略逊一筹，所以要依据具体的场景选择使用。

因为 LRUCache 才想一览 OD 的代码，有意思的是，在 OD 最初实现的 Issue 讨论中，就已经有人提出了支持 LRUCache 的想法，不过最终被否决了。

---

*References*

- [LRUCache - Leetcode](https://leetcode-cn.com/problems/lru-cache/)

- [OrderedDict vs dict - RealPython](https://realpython.com/python-ordereddict/#exploring-unique-features-of-pythons-ordereddict)

- [OrderedDict - Python Issue](https://bugs.python.org/issue5397)
