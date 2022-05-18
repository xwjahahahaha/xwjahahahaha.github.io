---
title: 剑指Offer45.把数组排成最小的数
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-19 00:39:47
---

## 题目描述

[剑指 Offer 45. 把数组排成最小的数](https://leetcode.cn/problems/ba-shu-zu-pai-cheng-zui-xiao-de-shu-lcof/)

难度中等

输入一个非负整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。

<!-- more --> 

**示例 1:**

```
输入: [10,2]
输出: "102"
```

**示例 2:**

```
输入: [3,30,34,5,9]
输出: "3033459"
```

 

**提示:**

- `0 < nums.length <= 100`

**说明:**

- 输出结果可能非常大，所以你需要返回一个字符串而不是整数
- 拼接起来的数字可能会有前导 0，最后结果不需要去掉前导 0

## 解题思路及代码

```go
// 排序规则实现
func minNumber(nums []int) string {
    // 排序
    quickSort(nums, 0, len(nums)-1)
    // 转换为字符串
    ans := ""
    for _, n := range nums {
        ans += strconv.Itoa(n)
    }
    return ans
}

// 改版的快排
func quickSort(nums []int, l, r int) {
    if l < r {
        mid := getMid(nums, l, r)
        quickSort(nums, l, mid)
        quickSort(nums, mid+1, r)
    }
}

func getMid(nums []int, l, r int) int {
    pivot := nums[l]
    for l < r {
        for l < r && compare(nums[r], pivot) { r-- }
        nums[l] = nums[r]
        for l < r && compare(pivot, nums[l]) { l++ }
        nums[r] = nums[l]
    }
    nums[l] = pivot
    return l
}

func compare(a, b int) bool {
    x, y := strconv.Itoa(a), strconv.Itoa(b)
    if strPlus(x, y) >= strPlus(y, x) {     // 注意相等条件也要为true
        // x+y >= y+x 则x排在前面, 反之亦然
        return true
    }
    return false
}

func strPlus(a, b string) int {
    sum, _ := strconv.Atoi(a+b)
    return sum
}
```

