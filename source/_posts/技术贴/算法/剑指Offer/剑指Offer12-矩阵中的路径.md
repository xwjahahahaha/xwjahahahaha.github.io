---
title: 剑指Offer12.矩阵中的路径
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-20 11:24:57
---

## 题目描述

[剑指 Offer 12. 矩阵中的路径](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/)

难度中等

给定一个 `m x n` 二维字符网格 `board` 和一个字符串单词 `word` 。如果 `word`存在于网格中，返回 `true` ；否则，返回 `false` 。

单词必须按照字母顺序，通过相邻的单元格内的字母构成，其中“相邻”单元格是那些水平相邻或垂直相邻的单元格。同一个单元格内的字母不允许被重复使用。

<!-- more --> 

例如，在下面的 3×4 的矩阵中包含单词 "ABCCED"（单词中的字母已标出）。

![Qehqko](http://xwjpics.gumptlu.work/qinniu_uPic/Qehqko.png)

 

**示例 1：**

```
输入：board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED"
输出：true
```

**示例 2：**

```
输入：board = [["a","b"],["c","d"]], word = "abcd"
输出：false
```

**提示：**

- `1 <= board.length <= 200`
- `1 <= board[i].length <= 200`
- `board` 和 `word` 仅由大小写英文字母组成

**注意：**本题与主站 79 题相同：https://leetcode-cn.com/problems/word-search/

## 解题思路及代码

## 解题思路

搜索问题 => BFS、DFS

每个步骤多个选择 => 树状DFS => 回溯法

回溯法优化 => 子问题一致性剪枝 => 一旦当前位置字母不符, 立即返回false, 不向下递归

回溯一般模版:

```go
// 子集一般对应递归树path
回溯(子集, 全集):
    if 满足条件/结束条件:  	// 下一步做什么,所以会当成结束条件
        加入答案
    for 元素 in 全集:			 // 这一步要做什么, 再过度到下一步
        元素加入子集
        回溯(子集, 全集)		// 回溯到上一步需要改回什么
        回退操作/元素退出子集
```

## 代码

```golang
// 回溯法
func exist(board [][]byte, word string) bool {
    m, n := len(board), len(board[0])
    // 已走过的标记
    var visited [][]bool
    for i:=1; i<=m; i++ {
        visited = append(visited, make([]bool, n))
    }
    var rollback func(int, int, int) bool
    rollback = func(i, j, p int) bool {
        // 当前位置
        // 结束情况:
        // 1. 当前位置不相同直接返回false, 结束回溯
        if board[i][j] != word[p] {
            return false
        }
        // 2, 当p为最后一个直接返回true
        if p == len(word)-1 {
            return true
        }
        // 当前位置标记为访问过
        visited[i][j] = true
        // 下一位置
        // 上
        if i > 0 && !visited[i-1][j] {
            if rollback(i-1, j, p+1) {
                return true
            }
        } 
        // 下
        if i < m-1 && !visited[i+1][j] {
            if rollback(i+1, j, p+1) {
                return true
            }
        }
        // 左
        if j > 0 && !visited[i][j-1] {
            if rollback(i, j-1, p+1){
                return true
            }
        }
        // 右
        if j < n-1 && !visited[i][j+1] {
            if rollback(i, j+1, p+1){
                return true
            }
        }
        // 上一位置: 回溯
        visited[i][j] = false
        return false
    }
    // 调用回溯函数, 寻找合适的入口
    for i:=0; i<m; i++ {
        for j:=0; j<n; j++ {
            if rollback(i, j, 0) {
                return true
            }
        }
    }
    return false
}
```

## 复杂度分析

### 时间复杂度

设word字符串长度为K, board矩阵长宽分别为m, n

那么时间复杂度就是$O(3^K*M*N)$

* rollback回溯函数: 每一个字符除去上一步的方向还剩三个方向, 一共就是$3^K$的可能性,

* 外层两个for循环提供M*N的复杂度

### 空间复杂度

这里采用额外空间保护原数组, 所以时间复杂度为$O(M*N)$

还可采用标记的方法(例如将已经访问的位置字符变为“\*”)降低时间复杂度为递归栈的空间K(其最差也为M*N)

## 编码注意事项/思路

1. 每个方向中不要直接返回下一步递归的结果, 因为**某一个方向为false 不代表当前整个path都是false, 题意是存在,所以只要有一个方向是true则即继续**

2. 调用rollback回溯函数与上方同理,全部为false才返回false

   
