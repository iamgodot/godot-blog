---
title: "Overflow in Python"
date: 2021-11-12T17:20:10+08:00
categories:
  - Code
draft: false
---

犹记得在 Java 中，int 占用 4 bytes 所以大小限制在 -2^32 和 2^32 -1 之间，如果考虑到溢出的情况，就要用 long 类型来转换。

但是在 Python 中似乎从来没有考虑过类似的问题，也不记得 int 是占几个字节，那么是说 Python 的数字永远不会溢出吗？不可能吧。

答案是对于 int 类型来说，可以认为不会；而 float 类型则需要注意溢出的情况。

首先说 int 类型，打印 `sys.maxsize` 可以看到 Python 会根据系统是 32 位还是 64 位来确定一个上限值，这点与 Java 等语言一致。不同的是我们仍然可以使用比这个上限大（得多）的整数，因为 Python 支持的是 [Arbitrary-precision arithmetic](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic)。也就是说为了突破 CPU 字长带来的运算限制，通过在软件层面模拟计算过程，是可以完成更高位数和精度的运算的。比如在公钥加密的场景下，经常需要对上百位的整数进行运算，这时候就需要在软件层面支持。

这确实说明我们可以使用任意长度的 int 数字，只要不超出内存限制的话。因为如果给 Python 解释器分配了 2GB 内存，但是 2 * 1024 * 1024 * 1024 * 8 这么多位也不够表达的话，还是会出错的，只是并非 Overflow，而是 [MemoryError](https://docs.python.org/3/library/exceptions.html#MemoryError).

再说 float. 打印 `sys.float_info.max` 可以看到 float 的上限值，如果超出之后是会报错的，即 [OverflowError](https://docs.python.org/3/library/exceptions.html#OverflowError). 可以这么来测试：

```python
In [9]: m = 2 ** 100000

In [10]: m*.1
---------------------------------------------------------------------------
OverflowError                             
Traceback (most recent call last) <ipython-input-10-03cfc6457eba> in <module>
----> 1 m*.1

OverflowError: int too large to convert to float
```

另外不要把这个上限与 float infinity 搞混了，因为在 Python 中可以用 `float('inf')` 来表示无限大，但它并非具体值，而是概念上的定义。如果非要比较的话，`sys.float_info.max < float('inf')` 是会返回 True 的。

---

*References*

- [Arbitrary-precision arithmetic - Wikipedia](https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic)
- [OverflowError - Python](https://docs.python.org/3/library/exceptions.html#OverflowError)
- [Maxsize - Python](https://docs.python.org/3/library/sys.html#sys.maxsize)
- [How is there no integer overflow in Python? - Quora](https://www.quora.com/How-is-there-no-integer-overflow-in-Python)