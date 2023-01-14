---
title: 剑指Offer53-I.在排序数组中查找数字I
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-07-08 22:10:06
---

## 题目描述

[剑指 Offer 53 - I. 在排序数组中查找数字 I](https://leetcode.cn/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/)

难度简单

统计一个数字在排序数组中出现的次数。

**示例 1:**

```
输入: nums = [5,7,7,8,8,10], target = 8
输出: 2
```

**示例 2:**

```
输入: nums = [5,7,7,8,8,10], target = 6
输出: 0
```

 <!-- more -->

**提示：**

- `0 <= nums.length <= 105`
- `-109 <= nums[i] <= 109`
- `nums` 是一个非递减数组
- `-109 <= target <= 109`

**注意：**本题与主站 34 题相同（仅返回值不同）：https://leetcode-cn.com/problems/find-first-and-last-position-of-element-in-sorted-array/

## 解题思路及代码

```go
// 二分找位置
// 最坏时间复杂度O(N)(全为一个数)
func search(nums []int, target int) int {
    count := 0
    l, r := 0, len(nums)
    for l < r {
        mid := l+(r-l)>>1
        if nums[mid] == target {
            count++
            // 左右拓展
            for i:=mid-1; i>=0 && nums[i]==target; i-- {
                count++
            }
            for i:=mid+1; i<len(nums) && nums[i]==target; i++ {
                count++
            }
            break
        }else if nums[mid] < target {
            l = mid+1
        }else {
            r = mid
        }
    }

    return count
}

// 更好的思路：二分找到两个位置：
// left: >=target的第一个数
// right: >target的第一个数
// 结果 = right-left
func search(nums []int, target int) int {
    lval, rval := binarySearch(nums, target, true), binarySearch(nums, target, false)-1
    if lval <= rval && rval < len(nums) && nums[lval] == target && nums[rval] == target {
        return rval-lval+1
    }
    
    return 0
}

// binarySearch 二分查找,lower为true表示>=target的第一个数, 返回目标位置idx
func binarySearch(nums []int, target int, lower bool) int {
    l, r, idx := 0, len(nums), len(nums)
    for l < r {
        mid := l + (r-l) >> 1
        if nums[mid] > target || (lower && nums[mid] >= target){
            r = mid
            idx = mid
        }else {
            l = mid+1
        }
    }

    return idx
}
```

二刷：

```go
func search(nums []int, target int) int {
    l, r := binarySearch(nums, target, true), binarySearch(nums, target, false)
    if l == r {
        return 0
    }

    return r-l
}

func binarySearch(nums []int, target int, lower bool) int {
    l, r := 0, len(nums)
    res := len(nums)

    for l < r {
        mid := (r-l)/2 + l
        if nums[mid] > target || (lower && nums[mid] >= target) {
            r = mid
            res = mid
        }else {
            l = mid+1
        }
    }

    return res
}
```

