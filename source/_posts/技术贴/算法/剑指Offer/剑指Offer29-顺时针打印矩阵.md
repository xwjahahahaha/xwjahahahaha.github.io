---
title: 剑指Offer29.顺时针打印矩阵
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-20 13:53:39
---

## 题目描述

[剑指 Offer 29. 顺时针打印矩阵](https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/)

难度简单

输入一个矩阵，按照从外向里以顺时针的顺序依次打印出每一个数字。

 

**示例 1：**

```
输入：matrix = [[1,2,3],[4,5,6],[7,8,9]]
输出：[1,2,3,6,9,8,7,4,5]
```

**示例 2：**

```
输入：matrix = [[1,2,3,4],[5,6,7,8],[9,10,11,12]]
输出：[1,2,3,4,8,12,11,10,9,5,6,7]
```

 

**限制：**

- `0 <= matrix.length <= 100`
- `0 <= matrix[i].length <= 100`

注意：本题与主站 54 题相同：https://leetcode-cn.com/problems/spiral-matrix/


<!-- more -->

## 解题思路及代码

```go
// 直接的思路
// 时间复杂度O(N*M) 空间复杂度O(N*M)
func spiralOrder(matrix [][]int) []int {
    m := len(matrix)
    if m == 0 {
        return []int{}
    }
    n := len(matrix[0])
    flagMatrix := make([][]bool, 0)     // 已访问标记数组
    for i:=0; i<m;i++ {
        flagMatrix = append(flagMatrix, make([]bool, n))
    }
    path, dire := []int{}, 1    // 1: 右, 2: 左, 3: 上, 4:下
    for i, j := 0, 0; ;{
        // 向右走
        if dire == 1 {
            if i < 0 || i >= m || j < 0 || j >= n || flagMatrix[i][j] {
                break
            }
            path = append(path, matrix[i][j])
            flagMatrix[i][j] = true
            j++
            if j == n || flagMatrix[i][j] {
                dire = 4
                j--
                i++
                continue
            }
        }
        // 向下走
        if dire == 4 {
            if i < 0 || i >= m || j < 0 || j >= n || flagMatrix[i][j] {
                break
            }
            path = append(path, matrix[i][j])
            flagMatrix[i][j] = true
            i++
            if i == m || flagMatrix[i][j] {
                dire = 2
                i--
                j--
                continue
            }
        }
        // 向左走
        if dire == 2 {
            if i < 0 || i >= m || j < 0 || j >= n || flagMatrix[i][j] {
                break
            }
            path = append(path, matrix[i][j])
            flagMatrix[i][j] = true
            j--
            if j == -1 || flagMatrix[i][j] {
                dire = 3
                j++
                i--
                continue
            }
        }
        // 向上走
        if dire == 3 {
            if i < 0 || i >= m || j < 0 || j >= n || flagMatrix[i][j] {
                break
            }
            path = append(path, matrix[i][j])
            flagMatrix[i][j] = true
            i--
            if i == -1 || flagMatrix[i][j] {
                dire = 1
                i++
                j++
                continue
            }
        }
    }
    return path
}
```

