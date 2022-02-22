---
title: 剑指Offer03.数组中重复的数字
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-05-26 16:26:24
---

## 题目描述

[剑指 Offer 03. 数组中重复的数字](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

难度简单

找出数组中重复的数字。

<!-- more -->
在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

**示例 1：**

```
输入：
[2, 3, 1, 0, 2, 5, 3]
输出：2 或 3  
```

**限制：**

```
2 <= n <= 100000
```

## 解题思路及代码

```go
// 重复次数 => 散列表, 给定范围 => 数组实现散列表
// 时间复杂度O(n), 空间复杂度O(n)
func findRepeatNumber(nums []int) int {
    n := len(nums)
    account := make([]int, n)
    for _, v := range nums {
        account[v] += 1
        if account[v] > 1 {
            return v
        }
    }
    return -1
}

// “所有数字都在0~n-1的范围内” => 如果无重复则排序后的数组第i位数字一定是i
// 如果当前位置i的数v不相同, 那么判断nums[v]位置的值:
// 如果v==nums[v], 那么出现重复,结束
// 如果不等, 那么交换两者,使v在正确的位置(与下标相同)
// 对当前位置重复上述操作, 直到i == v
// 时间复杂度O(n), 空间复杂度O(1)
func findRepeatNumber(nums []int) int {Ω
    for i, v := range nums {
        // 每个位置最多交换两次就满足
        for i != v{
            if v == nums[v] {
                return v
            }else {
                nums[i], nums[v] = nums[v], v
            }
        }
    }
    return -1
}
```

