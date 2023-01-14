---
title: 剑指Offer53-II.0～n-1中缺失的数字
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-07-08 22:58:56
---

## 题目描述

[剑指 Offer 53 - II. 0～n-1中缺失的数字](https://leetcode.cn/problems/que-shi-de-shu-zi-lcof/)

难度简单

一个长度为n-1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0～n-1之内。在范围0～n-1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。

 <!-- more -->

**示例 1:**

```
输入: [0,1,3]
输出: 2
```

**示例 2:**

```
输入: [0,1,2,3,4,5,6,7,9]
输出: 8
```

 

**限制：**

```
1 <= 数组长度 <= 10000
```

## 解题思路及代码

```go
// 二分查找nums[i] != i 的数
// 空间复杂度O(1)
// 时间复杂度O(logN)
func missingNumber(nums []int) int {
    l, r, res := 0, len(nums)-1, len(nums)-1
    for l <= r {
        mid := l + (r-l)>>1
        if nums[mid] == mid {
            // 舍弃前半段,因为前半段都是满足条件的了
            l = mid+1
        }else {
            // 找到错位的位置，继续向前寻找, 因为可能不是第一个
            r = mid-1
            res = mid
        }
    }
    
    if l == len(nums) {
        // l已经在此处返回了，说明整个数组都是有序满足条件的，结果就是最后一个没出现的值即n
        return len(nums)
    }

    return res
}

// 异或
// 空间复杂度O(2N)
// 时间复杂度O(N)
func missingNumber(nums []int) int {
    n := len(nums)
    tmp := make([]int, 2*n+1)
    for i:=0; i<2*n+1; i++ {
        if i <= n-1 {
            tmp[i] = nums[i]
        }else {
            tmp[i] = i-n
        }
    }

    // 因为少的数只出现了一次，所以异或的结果就是该值
    res := 0
    for i:=0; i<2*n+1; i++ {
        res ^= tmp[i]
    }

    return res
}
```

二刷

```go
// 异或, 异或0~n-1的数组，然后再亦或原本缺一个的数组就是答案
func missingNumber(nums []int) int { 
    res := 0
    n := len(nums)
    for i:=0; i<=n; i++ {
        res ^= i
    }

    for _, v := range nums {
        res ^= v
    }

    return res
}
```

