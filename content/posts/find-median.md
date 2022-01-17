---
title: "Find Median"
date: 2021-11-03T17:07:04+08:00
categories:
  - Code
tags:
  - dsa
draft: false
---

祸从天降的一天。

早上起不来，于是刷手机清醒一下，突然看到一个 ACMer 楼主提到自己没有刷过 Leetcode，面试的时候差点儿被打脸了。

看了一下题目，要求是 O(logn) 的复杂度，默默想了想，没有特别清晰的思路。

结果翻了翻评论，很多人都在蜻蜓点水般地表示二分查找不断分割就可以了。

要那么简单还用你说吗？起床了起床了。

题目是这样的：

> 给定两个大小分别为 m 和 n 的正序（从小到大）数组 nums1 和 nums2。请你找出并返回这两个正序数组的中位数。

我本来的想法是归并之后计算中位数，但只能做到 O(m+n)，再优化感觉只能二分了。

于是又开始想分别取两个数组的中位数，比较之后就可以各扔掉一半，然后对两个折半的数组继续取中位数比较。

比如一开始找到的中位数是这样：[...5...], [...10...]，那么 5 的左边和 10 的右边就可以丢掉了，因为最终的中位数肯定在 5 和 10 中间。

如果接下来是这样：[5..8...], [...7..10]，那么 8 的右边和 7 的左边也可以丢掉，因为比 8 大的元素数量达不到这两个小数组的一半，所以中位数不会在里面，比 7 小的同理。

一直重复这个流程，最后得到的肯定就是一个或两个数，平均下就好了，时间复杂度也是符合要求的 O(logn).

结果等到看完标答后我才反应过来自己错在哪里了，这是后话。（因为 m 和 n 不一定一样大，所以显然折半的逻辑不对）

最优的解决方案是可以做到 O(log min(m, n)) 的，核心也是二分法，不过思路要复杂一些：

1. 首先明确中位数的定义。对于奇数个数字来说是最中间的元素，偶数则是中间两个元素的平均值。
2. 要做的是将两个数组各分一刀，假设划分的下标分别是 i 和 j，那么 nums1 被分成 nums1[0, i], nums1[i:]，而 nums2 分为 nums2[0, j], nums2[j:].
3. 因为数组中元素的数量是奇是偶不一定，所以规定好是奇数的话多的一个放到左边。如此 i 的位置就是右边部分的第一个，而左半边正好有 i 个元素。
4. 下标 i 和 j 之间是存在一个等式关系的，因为是中位数，所以 i + j 等于元素总数的一半，或者一半多一个（因为说好了左边多一个嘛）。那么就有 i+j=(m+n)/2 | i+j=(m+n)/2+1，合并起来写成 (m+n+1)/2.
5. 一开始 i 的选择就先定在 nums1 的中间位置，然后根据规则不断地二分，如此就完成了最核心的循环。那么根据什么规则呢？就是两个左半部分不能大于两个右半部分，也就是 nums1[i-1]<=nums2[j] & nums2[j-1]<=nums1[i].
6. 还要注意的是 i 不是每次向左或向右移动一位，而是要按照二分的规则走，不然就变成线性的复杂度了。
7. 解决了核心逻辑，还有两个问题要考虑：
        1. 中位数计算。走完了循环我们可以获得四个数值，也就是 i-1, i, j-1, j 对应的四个元素。
               1. 如果元素总数是奇数的话，我们比较 nums1[i-1] 和 nums2[j-1] 取个大的就好了（还是那句话，说好了左边多一个嘛）。
               2. 否则就要取中间两个数平均了。这两个数就是两个左边的最大值，和两个右边的最小值。
        2. 边界条件。之所以后说这个是因为中位数计算的时候不考虑边界就会报错，比如对 i 来说，如果 i=0 或者 i=m 的话都是会出界的，所以要分别处理。
               1. 如果 i=0，说明 nums1 的左半部分为空，那么就假设这里的最大值为无限小，这样比较的话就相当于只考虑 nums2 的左半部分了。
               2. 如果 i=m，说明 nums1 的右半部分为空，那么就假设这里的最小值为无限大，这样比较的话就相当于只考虑 nums2 的右半部分了。

上代码：

```python
def find_median(nums1, nums2):
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1
    m, n = len(nums1), len(nums2)
    left, right = 0, m
    total_left = (m + n + 1) // 2  # 注意用 // 而不是 /，因为下标不能是浮点数
    while left < right:
        i = left + (right - left + 1) // 2  # 注意用 // 而不是 /，因为下标不能是浮点数
        j = total_left - i
        if nums1[i - 1] > nums2[j]:
            right = i - 1
        else:
            left = i

    i, j = left, total_left - left
    first_left_max = nums1[i - 1] if i > 0 else -float("inf")
    first_right_min = nums1[i] if i < m else float("inf")
    second_left_max = nums2[j - 1] if j > 0 else -float("inf")
    second_right_min = nums2[j] if j < n else float("inf")

    if (m + n) % 2 == 1:
        return max(first_left_max, second_left_max)
    else:
        return (
            max(first_left_max, second_left_max)
            + min(first_right_min, second_right_min)
        ) / 2

```

用 Python 实现的话，有两点需要注意：

1. 计算 `total_left` 和 `i` 的时候要用 `//`.
2. 使用 `float("inf")` 来表示极大值，加负号表示极小值。

到了这里基本就结束了，但是仔细看会发现函数的一开始会比较两个数组的长度，进而保证 nums1 是长度更短的那一个。

这是因为如果 nums1 非常长而 nums2 很短会造成 `j = total_left - i` 计算出来的 j 超过 nums2 的长度而出界。

而这是因为 i 是根据 left 和 right 计算得到的（也就是 m），从而能保证界限，j 是减出来的就不一定保险了。

最后来看一下这种方法的复杂度，因为保证了 nums1 较短，所以二分得到的时间复杂度为 O(logmin(m, n))，空间复杂度 O(1).

虽说上面的解法最优，但并不是一个很通用的方案，可以找到中位数，但对于求 k 位数这种问题就解决不了了。又看了下官方次优的解法，虽然时间复杂度为 O(log(m+n))，但是更普适：

1. 分别找出两个数组的 k/2-1 的位置，那么这个位置前面有 k/2-1 个元素。
2. 两个位置上的元素分别成为 pivot1, pivot2，如果 pivot1 的值小于 pivot2 的值，则可以舍弃 pivot1 及前面的 k/2-1 个元素。（因为即使 pivot2 前面的元素都小于 pivot1，pivot1 最多也就是第 k-1 大的元素，中位数肯定不在其中）
4. 去掉了这部分元素之后 nums1 就相当于变短了，而 nums2 不变，于是基于这两个数组继续。
5. 同时也要记得更新 k 值，因为已经去掉一部分元素了，所以 k 变为了原来的一半。
6. 继续此流程。当然过程中还要注意空数组、下标越界和 k=1 等边界情况。

~~此处不贴代码了，见官方答案。~~ 实现代码如下：

```python
def find_median(nums1, nums2):
    def find_kth_element(k, nums1, nums2):
        '''注意 k 是从 1 开始而不是 0'''
        m, n = len(nums1), len(nums2)
        start1, start2 = 0, 0
        while True:
            if start1 == m:
                return nums2[start2 + k - 1]
            if start2 == n:
                return nums1[start1 + k - 1]
            if k == 1:
                return min(nums1[start1], nums2[start2])
            # 因为 if 检查在上面，所以要取 min 防止下面 nums1[i] 或 nums2[j] 越界
            i = min(start1 + k // 2 - 1, m - 1)
            j = min(start2 + k // 2 - 1, n - 1)
            if nums1[i] <= nums2[j]:
                # k 的更新基于 start1，所以先改 k 再改 start1
                k -= i - start1 + 1
                start1 = i + 1
            else:
                k -= j - start2 + 1
                start2 = j + 1

    length = len(nums1) + len(nums2)
    if length % 2 == 1:
        return find_kth_element(length // 2 + 1, nums1, nums2)
    else:
        return (
            find_kth_element(length // 2, nums1, nums2)
            + find_kth_element(length // 2 + 1, nums1, nums2)
        ) / 2
```

---

整理完思路和代码已经是下午了，疲惫地什么都不想做。刷手机须谨慎，有风险别乱看。

---

*References*

- [Media of two sorted arrays 官方答案 - Leetcode](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/solution/xun-zhao-liang-ge-you-xu-shu-zu-de-zhong-wei-s-114/)
- [个人觉得比较容易理解的答案 - Leetcode](https://leetcode-cn.com/problems/median-of-two-sorted-arrays/solution/he-bing-yi-hou-zhao-gui-bing-guo-cheng-zhong-zhao-/)
