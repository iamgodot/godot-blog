---
title: "Numeric Strings in Python"
date: 2021-11-21T17:44:36+08:00
categories:
  - Code
draft: false
---

Python 的字符串自带了三种判断字符是否为数字的方法，但实际用处却相差很多。

# TL;DR

- 三种方法 isdecimal < isdigit < isnumeric，即包含的范围越来越大
- 除了 ASCII 字符以外，对于 Unicode 的字符也都覆盖在内
- 三种方法对于小数点和负号都会返回 False
- 三种方法对于空字符串都会返回 False
- 比较简便判断数字字符串的方法：直接使用 float 方法并检测 ValueError

# Decimal & Digit & Numeric

对于 isdecimal, isdigit 和 isnumeric 三种方法，目的并不是判断字符串是不是一个有效数字，而是针对每一个字符的校验：

- isdecimal: 判断字符串中的字符是否都为 Decimal，也就是在 Unicode 中类别为 Nd 的字符
- isdigit: 除了 isdecimal 包含的范围之外，还会判断字符是否都为 Digit，即 Unicode 的 Numeric_Type 为 Digit 或 Decimal
- isnumeric: 除了 isdigit 包含的范围之外，还会判断字符是否都为 Numeric，即 Numeric_Type 为 Numeric

所以这三种方法覆盖的字符范围，每一个都是前一个的超集。对于超出 ASCII 字符之外的效果，比如：

- "０１２３４５６７８９" 这种 Full-width 字符串 isdecimal 会判定为 True，后两个方法也一样
- "⓪①②③④⑤⑥⑦⑧⑨" 这种 Circled-digit 字符串 isdecimal 判定 False，但 isdigit 和 isnumeric 为 True
- "一二三四五六七八九十壹貳參肆伍陸柒捌玖拾" 这种中文数字字符串只有 isnumeric 才会判定为 True

总之这几种方法有更广泛的用途，根本不是为了简单的 ASCII 数字字符串的判断。即使用来做判断的话，局限性也非常大，因为如果包含小数点和负号，三个方法都会返回 False.

# More Methods

另外 Str 还有 isalpha, isalnum 和 isascii 三种方法：

- isalpha: 判断的字符在 Unicode 中是不是 Letter 类型，除了英文字母，其他语言比如中文汉字也都会判定为 True
- isalnum: 这个比较简单，相当于判断 isnumeric or isalpha
- isascii: 判断是否为 ASCII 字符，但是对于空字符串是返回 True 的，而上面的方法都会返回 False

有一种比较常见的需求是判断字符串是否非空，那么直接 `if bool(foo)` 或者 `if foo` 即可。

# Use float()

那么有什么方法可以判断字符串是否为数字呢？这个需要先确定数字的范围，比如是否包含小数、负数、指数等等。

使用 float 是个比较简便的方法：

```python
def is_number(foo):
    try:
        float(foo.strip().replace(',', '').replace(' ', ''))
    except ValueError:
        return False

    return True
```

注意 float 对于 `"inf"` 会返回 `inf` 这个浮点数上限，在 Python 中会利用这个值做一些有用的数值比较。

另外 float 类型的变量还有一个 is_integer 方法，用来判断整数很方便。

# Valid Number

Leetcode 上面有[一道判断有效数字的题目](https://leetcode-cn.com/problems/valid-number/)是一个很好的练习，不仅需要考虑小数、指数和正负号，甚至 e 的大小写，Python float 中的 `"inf"` 和 `"infinity"` 都考察了。

使用纯粹的条件判定虽然有些繁琐，但也并不是很困难。不过用正则或者有限状态机来解是更聪明的做法。

状态机的做法单独整理一篇好了，这里略过不提。

---

*References*

- [Unicode numeric values and types - Wikipedia](https://en.wikipedia.org/wiki/Unicode_character_property#Numeric_values_and_types)
- [Isdecimal - Python](https://docs.python.org/3/library/stdtypes.html#str.isdecimal)
- [Isdigit - Python](https://docs.python.org/3/library/stdtypes.html#str.isdigit)
- [Isnumeric - Python](https://docs.python.org/3/library/stdtypes.html#str.isnumeric)
- [Check if a string is numeric...](https://note.nkmk.me/en/python-str-num-determine/)
- [What's the difference between str.isdigit, is numeric and isdecimal in python - Stackoverflow](https://stackoverflow.com/questions/44891070/whats-the-difference-between-str-isdigit-isnumeric-and-isdecimal-in-python)
