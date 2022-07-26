---
title: "Sum of Total Strength of Wizards"
date: 2022-07-26T10:41:55+08:00
draft: false
categories:
  - Code
tags:
  - dsa
keywords:
  - 巫师的总力量和
  - 单调栈
  - 前缀和
---

前两天有幸做了一道算法题，虽然没能成功解决，但题目很有意思，充分展现了算法的魅力所在。

抛开题面的包装不谈，核心内容就是给定一个数组，计算它的所有子数组的最小值与加和的乘积的总和。

（这里要注意子数组的定义，一定是连续的，如果不连续的话叫做子序列。）

比如对于 `[1, 2, 3]` 来说，一共有六种情况：

- `[1]`: `1 * 1 = 1`
- `[2]`: `2 * 2 = 4`
- `[3]`: `3 * 3 = 9`
- `[1, 2]`: `1 * (1 + 2) = 3`
- `[2, 3]`: `2 * (2 + 3) = 10`
- `[1, 2, 3]`: `1 * (1 + 2 + 3) = 6`

最后答案为 `1 + 4 + 9 + 3 + 10 + 6 = 33`。

# 暴力解法

直接的做法很容易想到，那就是嵌套遍历数组，对于每个子数组再计算 `min` 与 `sum` 的乘积，最后求和。

看着感觉还好？但其实除了双层循环之外，计算最小值以及求和也需要一次遍历，最终的时间复杂度为 `O(n^3)`，非常之慢。

看上去每个子数组的计算之间有很多重复，如何优化呢？

# 单调栈

单独来看每一个元素：在某个区间内它一定是最小值，即便只有它自己，那么此范围内的子数组的最小值直接使用该元素即可。

那么第一步就是找到每个元素的有效区间，于是问题变成了求前一个和后一个的更小元素。

单调栈是专门来解决这类问题的，简单来说就是在遍历过程中保证栈的单调性，进而为每个元素定位前/后的更小/大元素值。一般可以这么写：

```python
def find_last_greater(nums):
    stack = []
    length = len(nums)
    last_greater_element_index = [-1] * length
	for i in range(length):
    	while stack and nums[stack[-1]] <= nums[i]:
            stack.pop()
        if stack:
            last_greater_element_index[i] = stack[-1]
        stack.append(i)
    return last_greater_element_index
```

其中，遍历的顺序决定往前找（正序）还是往后找（倒序），与栈顶元素的比较决定找到的是更小还是更大的元素（的下标）。需要注意比较中的等号，它表示寻找的条件是严格单调，即排除了等值的情况。

此外，还有一个省力的技巧。如果想同时寻找前后两个更大的元素，那么只需遍历一次即可：

```python
def mono_stack(nums):
    stack = []
    length = len(nums)
    last_greater_element_index = [-1] * length
    next_greater_element_index = [length] * length
	for i in range(length):
    	while stack and nums[stack[-1]] <= nums[i]:
            # 这里正好利用出栈的机会来更新 next 数组
            next_greater_element_index[stack.pop()] = i
        if stack:
            last_greater_element_index[i] = stack[-1]
        stack.append(i)
    return last_greater_element_index, next_greater_element_index
```

更小元素的话也是一样，改变比较条件就好了。另外结果中的两个数组正好是一开一闭的，不会出现对于等值情况的重复。

还有初始值，比如对于前一个更大元素最好预设为 `-1`，这样逻辑上保持统一，计算区间长度的时候也更方便。

# 前缀和 -> 前缀和

找到了区间，也解决了最小值的问题，要想想怎么求和了。

假设 `[l, r]` 之内包含着元素 `i`，这样产生的子数组数量（包含 `i`）为两边长度的乘积：`(i - l + 1) * (r - i + 1)`。

如何加和呢，我当时也卡在了这里，虽然利用前缀和可以快速得到某个子数组的和，但依次遍历这些子数组求和的复杂度仍然很高。

后来才知道，其实可以更上一层楼，应用前缀和的前缀和快速计算出区间内所有子数组和的总和。

公式推导的过程参考[这里](https://lctemplates.xyli.codes/en/latest/prefix-sum.html#prefix-sum-of-prefix-sum)，截图如下：

![](https://static.iamgodot.com/content/images/20220726162704.png)

一般的前缀和是前面所有元素加上当前元素之和，而图中的定义略有不同，每个前缀和并不包括当前位置的元素。基于此，推导图中第二行的右半部分应为 `presum[r] - presum[l - 1]`。

另外，第三行到第四行之所以消解求和符号是因为公式中没有再依赖该变量，所以能够转化为乘积关系。

# 实现

先以上图的原始推导结果展示代码：

```python
from itertools import accumulate

def total_strength(strength):
    length = len(strength)
    min_left, min_right = [-1] * length, [length] * length
    stack = []
    for i in range(length):
        while stack and strength[stack[-1]] >= strength[i]:
            min_right[stack.pop()] = i
        if stack:
            min_left[i] = stack[-1]
        stack.append(i)
    res = 0
    prepresum = list(accumulate(accumulate(strength, initial=0), initial=0))
    for i in range(length):
        range_sum = (i - min_left[i]) * (
            prepresum[min_right[i] + 1] - prepresum[i + 1]
        ) - (min_right[i] - i) * (prepresum[i + 1] - prepresum[min_left[i] + 1])
        res += range_sum * strength[i]
    return res % (10 ** 9 + 7)
```

可以看到，在计算前缀和数组时，先在最前面增加了一个 `0` 元素（通过 `initial` 参数），这样得到的前缀和就是不包含当前位置元素的。好处是不会造成数组越界，因为下面 `prepresum` 的下标存在 `i + 1`、`min_right[i] + 1` 等情况。

但这确实不符合直觉的前缀和定义，于是强迫症让我按照传统又推导了一遍，`range_sum` 中的 `+` 都变成了 `-`。这时候计算 `prepresum` 就不需要前面加 `0` 了。这里还有个坑要注意，因为 `-` 的存在，`prepresum` 的下标可能会小于 0，由于 Python 的切片特性并不会越界报错，而是从末尾继续计算，导致结果错误。因此，当下标变成负数时，value 要直接取 `0`。

```python
from itertools import accumulate

def total_strength(strength):
    length = len(strength)
    min_left, min_right = [-1] * length, [length] * length
    stack = []
    for i in range(length):
        while stack and strength[stack[-1]] >= strength[i]:
            min_right[stack.pop()] = i
        if stack:
            min_left[i] = stack[-1]
        stack.append(i)
    res = 0
    prepresum = list(accumulate(accumulate(strength)))
    for i in range(length):
        prepresum_i = prepresum[i - 1] if i > 0 else 0
        prepresum_left = prepresum[min_left[i] - 1] if min_left[i] - 1 >= 0 else 0
        prepresum_right = prepresum[min_right[i] - 1] if min_right[i] - 1 >= 0 else 0
        range_sum = (i - min_left[i]) * (prepresum_right - prepresum_i) - (min_right[i] - i) * (prepresum_i - prepresum_left)
        res += range_sum * strength[i]
    return res % (10 ** 9 + 7)
```

代码略微繁琐一点，不过我个人觉得还是更好理解些。

最后之所以对 `10^9 + 7` 取余是因为测试数据量很大，会造成溢出。

单调栈加前缀和的做法实现了线性的时间复杂度，相比暴力解法是飞跃般的提升，也让我切实感受到了算法之美。

---

*References*

- [巫师的总力量和 - Leetcode](https://leetcode.cn/problems/sum-of-total-strength-of-wizards/)：原题参考。
- [Prefix Sum](https://lctemplates.xyli.codes/en/latest/prefix-sum.html#prefix-sum-of-prefix-sum)：综合讲解前缀和，上面的公式推导即出自这里。
- [子数组的最小值之和 - Leetcode](https://leetcode.cn/problems/sum-of-subarray-minimums/)：另一道类似的题目，主要是针对单调栈的应用。









