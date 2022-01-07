---
title: 剑指Offer14-II.剪绳子II
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-07-14 14:18:34
---

## 题目描述

[剑指 Offer 14- II. 剪绳子 II](https://leetcode-cn.com/problems/jian-sheng-zi-ii-lcof/)

难度中等125

给你一根长度为 `n` 的绳子，请把绳子剪成整数长度的 `m` 段（m、n都是整数，n>1并且m>1），每段绳子的长度记为 `k[0],k[1]...k[m - 1]` 。请问 `k[0]*k[1]*...*k[m - 1]` 可能的最大乘积是多少？例如，当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到的最大乘积是18。

<font color='#e54d42'>答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。</font>

 <!-- more -->

**示例 1：**

```
输入: 2
输出: 1
解释: 2 = 1 + 1, 1 × 1 = 1
```

**示例 2:**

```
输入: 10
输出: 36
解释: 10 = 3 + 3 + 4, 3 × 3 × 4 = 36
```



**提示：**

- `2 <= n <= 1000`

注意：本题与主站 343 题相同：https://leetcode-cn.com/problems/integer-break/

## 解题思路及代码

循环求余法除去溢出

```go
// 此题不在适合动态规划，要用贪心算法
func cuttingRope(n int) int {
    if n == 1 || n == 2 {
        return 1
    } else if n == 3 {
        return 2
    } 

    mod := 1000000007
    timesOf3 := n/3
    if n- (3 * timesOf3) == 1 {
        timesOf3 -= 1
    }

    timesOf2 := (n - 3 * timesOf3) / 2

    // 求指数的时候，进行循环求模处理
    res := 1
    for i:=1; i<=timesOf3; i++ {
        res = (res * 3) % mod
    }
    for i:=1; i<=timesOf2; i++ {
        res = (res * 2) % mod
    }
    return res % mod
}
```

