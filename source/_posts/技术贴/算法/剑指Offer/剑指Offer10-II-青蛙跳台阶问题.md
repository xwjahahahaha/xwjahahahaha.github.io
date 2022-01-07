---
title: 剑指Offer10-II.青蛙跳台阶问题
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-16 11:31:08
---

## 题目描述

[剑指 Offer 10- II. 青蛙跳台阶问题](https://leetcode-cn.com/problems/qing-wa-tiao-tai-jie-wen-ti-lcof/)

难度简单169

一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶。求该青蛙跳上一个 `n` 级的台阶总共有多少种跳法。

答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。

**示例 1：**

```
输入：n = 2
输出：2
```

**示例 2：**

```
输入：n = 7
输出：21
```

**示例 3：**

```
输入：n = 0
输出：1
```

**提示：**

- `0 <= n <= 100`

注意：本题与主站 70 题相同：https://leetcode-cn.com/problems/climbing-stairs/

<!-- more -->

## 解题思路及代码

```go
// 对于N级台阶: 跳到第N级台阶的可能性次数 = 跳到第N-1台阶的可能性次数 (再跳一个台阶) + 跳到第N-2台阶的可能性次数 (再跳两个台阶)
// 即, f(N) = f(N-1) + f(N-2)
func numWays(n int) int {
    if n == 0 || n == 1 {
        return 1
    }
    x1, x2, sum := 1, 1, 0
    for i:=1; i<=n-1; i++ {
       sum = (x1 + x2) % 1000000007
       x1, x2 = x2, sum
    }
    return sum
}
```

