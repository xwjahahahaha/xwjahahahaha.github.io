---
title: 剑指Offer21.调整数组顺序使奇数位于偶数前面
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-03 10:44:13
---

## 题目描述

[剑指 Offer 21. 调整数组顺序使奇数位于偶数前面](https://leetcode-cn.com/problems/diao-zheng-shu-zu-shun-xu-shi-qi-shu-wei-yu-ou-shu-qian-mian-lcof/)

难度简单151收藏分享切换为英文接收动态反馈

输入一个整数数组，实现一个函数来调整该数组中数字的顺序，使得所有奇数位于数组的前半部分，所有偶数位于数组的后半部分。

<!-- more --> 

**示例：**

```
输入：nums = [1,2,3,4]
输出：[1,3,2,4] 
注：[3,1,2,4] 也是正确的答案之一。 
```

**提示：**

1. `0 <= nums.length <= 50000`
2. `1 <= nums[i] <= 10000`

## 解题思路及代码

```go
// 快速排序
// 时间复杂度
func exchange(nums []int) []int {
    if len(nums) == 0 {
        return nums
    }
    l, r := 0, len(nums)-1
    pivot := nums[l]
    for l < r {
        for l < r && (nums[r]%2 == 0) {
            r--
        }
         nums[l] = nums[r]
        for l < r && (nums[l]%2 == 1) {
            l++
        } 
        nums[r] = nums[l]
    }
    nums[l] = pivot
    return nums
}
```

