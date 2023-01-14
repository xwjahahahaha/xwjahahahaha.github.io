---
title: NC105.二分查找-II
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-12-01 22:35:49
---

## 题目描述

[NC105 二分查找-II](https://www.nowcoder.com/practice/4f470d1d3b734f8aaf2afb014185b395?tpId=188&&tqId=38588&rp=1&ru=/ta/job-code-high-week&qru=/ta/job-code-high-week/question-ranking)

可能有重复目标值的二分查找：找出第一个重复的数的位置

<!-- more -->

## 解题思路及代码

```go
package main

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 *
 * 如果目标值存在返回下标，否则返回 -1
 * @param nums int整型一维数组 
 * @param target int整型 
 * @return int整型
*/
// 改变一般二分法的思路，结束时找出的是刚好小于等于目标值的数
func search( nums []int ,  target int ) int {
    l, r := 0, len(nums)
    for l < r {
        mid := l + (r-l)>>1
        if nums[mid] < target {
            l = mid+1
        }else {
            r = mid
        }
    }
    // 结束时l在刚好不大于target的位置
    if l < len(nums) {        // 判断保证target存在，如果不满足则说明target一定不存在
        if nums[l] == target {    // 可能就是target
            return l
        }
        // 或者刚好比target小的数, 那么其下一个数一定就是target第一个位置的数
        if l < len(nums)-1 && nums[l+1] == target {
            return l+1
        }
    }
    return -1
}
```

二刷：

找到第一个>=目标值的数

```go
package main

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 *
 * 如果目标值存在返回下标，否则返回 -1
 * @param nums int整型一维数组 
 * @param target int整型 
 * @return int整型
*/
func search( nums []int ,  target int ) int {
    if len(nums) == 0 {
        return -1
    }
    
    l, r := 0, len(nums)
    res := len(nums)
    for l < r {
        mid := (r-l)/2 + l
        if nums[mid] >= target {
            r = mid
            res = mid
        }else {
            l = mid+1
        }
    }
    
    if nums[res] != target {
        return -1
    }
    return res
}
```

