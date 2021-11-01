---
title: "XOR"
date: 2021-10-28T22:57:41+08:00
categories: [Foundation]
tags: [logical operation ]
draft: false
---

# 异或运算

最近刷 leetcode 碰到了好几道异或运算（XOR）的题目，本文主要介绍异或运算的含义和应用。

## 什么是异或运算

异或运算，**Exclusive or** 或者 **exclusive disjunction** 是一个逻辑运算，当两个运算子不同（一个为真，另一个为假）的时候才为真。

我们都知道或运算（OR）的运算子只要有一个为真，结果就为真，这样就会有两种情况：

1.   一个为真，另一个为假
2.   两个都为真

这样 OR 的含义是不明确的，异或运算排除了第二种情况，异或这个名字也因此而来。

## 性质

### 基本运算

我们假定 `0` 为 `false`，`1` 为 `true`，XOR（使用 `^` 表示）的运算真值表如下：

| `x`  | `y`  | `x ^ y` |
| :--: | :--: | :-----: |
|  0   |  0   |    0    |
|  0   |  1   |    1    |
|  1   |  0   |    1    |
|  1   |  1   |    0    |

### XOR 和 0

任何一个数和 `0` XOR 的结果为自身

```
x ^ 0 = x
```

### XOR 和自身

任何一个数和自身 XOR 的结果为 `0`

```
x ^ x = 0
```

### 交换性

XOR 运算具有交换性

```
x ^ y = y ^ x
```

### 结合性

XOR 运算具有交换性

```
(x ^ y) ^ z = x ^ (y ^ z)
```

## 应用

### [交换值](https://www.geeksforgeeks.org/swap-two-numbers-without-using-temporary-variable/)

>   如何不使用额外空间快速的交换值

可以使用异或运算对两个变量连续进行三次 XOR 运算

```
x = x ^ y # =>                      (x ^ y, y)
y = x ^ y # => (x ^ y, y ^ x ^ y) = (x ^ y, x)
x = x ^ y # => (x ^ y ^ x, x)     = (y, x)
```

### [丢失的数字](https://leetcode-cn.com/problems/missing-number/)

>   给定一个包含 `[0, n]` 中 `n` 个数的数组 `nums` ，找出 `[0, n]` 这个范围内没有出现在数组中的那个数。

最快的方法就是把给的数组的所有成员（A[0] 一直到 A[n-2]）与 1 到 n 的数一起做 XOR 运算

```
A[0] ^ A[1] ^ ... ^ A[n-2] ^ 1 ^ 2 ^ ... ^ n
```

上面这个式子中，每个数组成员都会出现两次，相同的值进行异或运算就会得到 0。只有缺少的那个数字出现一次，所以最后得到的就是这个值。

类似的题目还有，其中找两个数的解题思路可以参考 Florian 大神的文章[^2]

-   [只出现一次的数字](https://leetcode-cn.com/problems/single-number/)
-   [只出现一次的数字 II](https://leetcode-cn.com/problems/single-number-ii/)
-   [只出现一次的数字 III](https://leetcode-cn.com/problems/single-number-iii/)
-   [寻找重复数](https://leetcode-cn.com/problems/find-the-duplicate-number/)

### 其他

除此之外，XOR 还有一些其他比较有意思的应用[^1]，包括但不限于

-   简化运算
-   加密
-   备份

[^1]:https://www.ruanyifeng.com/blog/2021/01/_xor.html
[^2]:https://florian.github.io/
