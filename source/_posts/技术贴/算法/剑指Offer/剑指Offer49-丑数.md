---
title: 剑指Offer49.丑数
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-29 15:27:24
---

## 题目描述

[剑指 Offer 49. 丑数](https://leetcode.cn/problems/chou-shu-lcof/)

难度中等

我们把只包含质因子 2、3 和 5 的数称作丑数（Ugly Number）。求按从小到大的顺序的第 n 个丑数。

 <!-- more -->

**示例:**

```
输入: n = 10
输出: 12
解释: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 是前 10 个丑数。
```

**说明:** 

1. `1` 是丑数。
2. `n` **不超过**1690。

注意：本题与主站 264 题相同：https://leetcode-cn.com/problems/ugly-number-ii/

## 解题思路及代码

```go
// 方法一：最小堆
// 时间复杂度O(nlogn)
// 空间复杂度O(n)
func nthUglyNumber(n int) int {
    onceMap := make(map[int]bool)
    minHeap := heap([]int{-1})
    minHeap.shiftUp(1)
    ans := 1
    for i:=0; i<n; i++ {
        // 1.出堆
        ans = minHeap.shiftDown()
        // 2.将其质因子倍数入堆
        if !onceMap[ans*2] {
            minHeap.shiftUp(ans*2)
            onceMap[ans*2] = true
        }
        if !onceMap[ans*3] {
            minHeap.shiftUp(ans*3)
            onceMap[ans*3] = true
        }
        if !onceMap[ans*5] {
            minHeap.shiftUp(ans*5)
            onceMap[ans*5] = true
        }
    }
    return ans
}

type heap []int

func (h *heap) shiftUp(v int) {
    *h = append(*h, v)
    i := len(*h)-1
    for i > 1 && (*h)[i] < (*h)[i/2] {
        (*h)[i], (*h)[i/2] = (*h)[i/2], (*h)[i]
        i /= 2
    }
}

func (h *heap) shiftDown() int {
    res := (*h)[1]
    (*h)[1] = (*h)[len(*h)-1]
    *h = (*h)[:len(*h)-1]
    i := 1
    n := len(*h)
    for {
        minIdx := i
        if i*2 < n && (*h)[i*2] < (*h)[minIdx] {
            minIdx = i*2
        }
        if i*2+1 < n && (*h)[i*2+1] < (*h)[minIdx] {
            minIdx = i*2+1
        }
        if i == minIdx {
            break
        }
        // 交换
        (*h)[i], (*h)[minIdx] = (*h)[minIdx], (*h)[i]
        i = minIdx
    }

    return res
}
```

```go
// 2. dp
// dp[i]表示第i个丑数
// 时间、空间复杂度O(N)
func nthUglyNumber(n int) int {
    dp := make([]int, n+1)
    // 初始化，第一个丑数就是1
    dp[1] = 1
    // p2指向当前还未*2的递增丑数序列的位置，如果一旦用了（就是其最小）此位置*2的数，那么就++；p3, p5同理
    p2, p3, p5 := 1, 1, 1
    // 填表
    for i:=2; i<=n; i++ {
        // 取三者最小，就是递增的下一个丑数
        dp[i] = min(min(dp[p2]*2, dp[p3]*3), dp[p5]*5)
        // 被使用的指针移动
        if dp[i] == dp[p2]*2 {
            p2++
        } 
        if dp[i] == dp[p3]*3 {
            p3++
        } 
        if dp[i] == dp[p5]*5 {
            p5++
        } 
    }
    return dp[n]
}

func min(a, b int) int {
    if a > b {
        return b
    }
    return a
}
```

