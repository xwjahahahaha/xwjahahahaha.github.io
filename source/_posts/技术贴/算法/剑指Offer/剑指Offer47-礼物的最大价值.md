---
title: 剑指Offer47.礼物的最大价值
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-20 00:12:23
---

## 题目描述

[剑指 Offer 47. 礼物的最大价值](https://leetcode.cn/problems/li-wu-de-zui-da-jie-zhi-lcof/)

难度中等

在一个 m*n 的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于 0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格、直到到达棋盘的右下角。给定一个棋盘及其上面的礼物的价值，请计算你最多能拿到多少价值的礼物？

 <!-- more -->

**示例 1:**

```
输入: 
[
  [1,3,1],
  [1,5,1],
  [4,2,1]
]
输出: 12
解释: 路径 1→3→5→2→1 可以拿到最多价值的礼物
```

提示：

- `0 < grid.length <= 200`
- `0 < grid[0].length <= 200`

## 解题思路及代码

```go
// dp
func maxValue(grid [][]int) int {
    if len(grid) == 0 {
        return 0
    }
    m, n := len(grid), len(grid[0])
    dp := make([][]int, m)
    for i:=0; i<m; i++ {
        dp[i] = make([]int, n)
    }
    // dp[i][j]表示走到此处的最大礼物价值
    // 初始化
    dp[0][0] = grid[0][0]
    for i:=1; i<m; i++ {
        dp[i][0] = dp[i-1][0] + grid[i][0]
    }
    for j:=1; j<n; j++ {
        dp[0][j] = dp[0][j-1] + grid[0][j]
    }
    // 填表
    for i:=1; i<m; i++ {
        for j:=1; j<n; j++ {
            dp[i][j] = max(dp[i-1][j], dp[i][j-1]) + grid[i][j]
        }
    }

    return dp[m-1][n-1]
}


func max(a, b int) int {
    if a < b {
        return b
    }
    return a
}
```

