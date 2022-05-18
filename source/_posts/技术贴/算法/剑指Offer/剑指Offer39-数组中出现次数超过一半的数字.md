---
title: Offer39.数组中出现次数超过一半的数字
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-01 10:02:53
---

## 题目描述

[剑指 Offer 39. 数组中出现次数超过一半的数字](https://leetcode-cn.com/problems/shu-zu-zhong-chu-xian-ci-shu-chao-guo-yi-ban-de-shu-zi-lcof/)

难度简单

数组中有一个数字出现的次数超过数组长度的一半，请找出这个数字。 

你可以假设数组是非空的，并且给定的数组总是存在多数元素。

 <!-- more -->

**示例 1:**

```
输入: [1, 2, 3, 2, 2, 2, 5, 4, 2]
输出: 2
```

 

**限制：**

```
1 <= 数组长度 <= 50000
```

 

注意：本题与主站 169 题相同：https://leetcode-cn.com/problems/majority-element/




## 解题思路及代码

摩尔投票法

```go
func majorityElement(nums []int) int {
    n := len(nums)
    if n == 0 {
        return -1
    }
    candidate, times := nums[0], 1
    for i:=1; i<n; i++ {
        if nums[i] != candidate {
            times--
        }else {
            times++
        }
        if times == 0 {
            candidate = nums[i]
            times = 1
        }
    }
    return candidate
}
```

