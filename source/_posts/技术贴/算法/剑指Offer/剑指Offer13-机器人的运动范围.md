---
title: 剑指Offer13.机器人的运动范围
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-26 10:26:11
---

## 题目描述

[剑指 Offer 13. 机器人的运动范围](https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/)

难度中等

地上有一个m行n列的方格，从坐标 `[0,0]` 到坐标 `[m-1,n-1]` 。一个机器人从坐标 `[0, 0] `的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），也不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格 [35, 37] ，因为3+5+3+7=18。但它不能进入方格 [35, 38]，因为3+5+3+8=19。请问该机器人能够到达多少个格子？

 <!-- more -->

**示例 1：**

```
输入：m = 2, n = 3, k = 1
输出：3
```

**示例 2：**

```
输入：m = 3, n = 1, k = 0
输出：1
```

**提示：**

- `1 <= n,m <= 100`
- `0 <= k <= 20`

## 解题思路及代码

### 解法一:递归深度优先遍历

#### 思路解析

* 递归基本思路： 对于(i, j)位置，其可达的最大格子数量 = 如果其自身位置可达，那么计算其可达的子问题(i+1, j), (i-1, j), (i, j+1), (i, j-1)位置的最大格子数量 再 + 1

  > 当所有的子问题被求出，当前位置也可得出

* 全局变量： 标记访问过的二维数组

* 题目隐含了优化条件： 从（0, 0）出发只往下、右走即可得到所有的格子。我们可以发现随着限制条件 k 的增大，(0, 0) 所在的蓝色方格区域内新加入的非障碍方格都可以由上方或左方的格子移动一步得到。而其他不连通的蓝色方格区域会随着 k 的增大而连通，且连通的时候也是由上方或左方的格子移动一步得到，因此我们可以将我们的搜索方向缩减为向右或向下。

  具体可见： https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/solution/ji-qi-ren-de-yun-dong-fan-wei-by-leetcode-solution/

#### 代码

```go
// DFS
func movingCount(m int, n int, k int) int {
    visited := [][]bool{}
    for i:=0; i<m; i++ {
        visited = append(visited, make([]bool, n))
    }
    return rollback(0, 0, k, m, n, visited)
}

func rollback(i, j, k, m, n int, visited [][]bool) int {
    // 当前位置/递归出口/结束情况
    if !checkNext(i ,j, k, m, n, visited) {
        // 一旦当前位置不可达，则返回0
        return 0
    }
    // 标记当前位置已访问
    visited[i][j] = true
    // 下一位置
    return 1 + rollback(i+1, j, k, m, n, visited) + rollback(i-1, j, k, m, n, visited) + rollback(i, j+1, k, m, n, visited) + rollback(i, j-1, k, m, n, visited)
  	// 优化
  	// return 1 + rollback(i+1, j, k, m, n, visited) + rollback(i, j+1, k, m, n, visited) 
}

// 判断当前位置是否可以走
func checkNext(i, j, k, m, n int, visited [][]bool) bool {
    if i > -1 && i < m && j > -1 && j < n && !visited[i][j] && (sumDigital(i) + sumDigital(j)) <= k {
        return true
    }
    return false
}

// 计算位数之和
func sumDigital(num int) int {
    sum := 0
    for num > 0 {
        sum += num % 10
        num /= 10
    }
    return sum
}
```

#### 编码注意

**不需要回退当前位置（即visited[i][j] = false）**，因为题目是求能够到达的最大格子数，所以一个格子访问过就标记上不用回退

一般回退是在矩阵搜索中的最长/优路径问题，将之前的路径回退

#### 复杂度分析

* 时间复杂度O(mn)，一共有m*n个状态要计算，每一个计算递归的时间复杂度为1
* 空间复杂度O(mn)

### 解法二：广度优先遍历（队列）

#### 思路解析

* BFS对于矩阵的遍历并不是很直观、友好，推荐还是DFS

* 将(i, j)所有相邻点(上下左右)加入队列，再不断的取出队列，计数

* 同样的，优化的解法可以只往下、右走，即只添加下、右节点入队

#### 代码

```go
// BFS
type local struct {
    i int
    j int
}
func movingCount(m int, n int, k int) int { 
    account := 0
    // 初始化全局访问数组
    visited := [][]bool{}
    for i:=0; i<m; i++ {
        visited = append(visited, make([]bool, n))
    }
    queue := []local{}
    // 初始化（0，0）节点
    initLocal := local{0, 0}
    queue = append(queue, initLocal)
    for len(queue) > 0 {
        // 弹出
        newLocal := queue[0]
        queue = queue[1:]
        // 处理
        // 当前位置
        if checkNext(newLocal, m, n, k, visited) {
            account ++
            visited[newLocal.i][newLocal.j] = true
            //入队
            queue = append(queue, local{newLocal.i + 1, newLocal.j})
            queue = append(queue, local{newLocal.i - 1, newLocal.j})
            queue = append(queue, local{newLocal.i, newLocal.j + 1})
            queue = append(queue, local{newLocal.i, newLocal.j - 1})
        } 
    }
    return account
}

// 判断当前位置是否可以走
func checkNext(l local, m, n, k int, visited [][]bool) bool {
    if l.i > -1 && l.i < m && l.j > -1 && l.j < n && !visited[l.i][l.j] &&(sumDigital(l.i) + sumDigital(l.j)) <= k {
        return true
    }
    return false
}

// // 计算位数之和
func sumDigital(num int) int {
    sum := 0
    for num > 0 {
        sum += num % 10
        num /= 10
    }
    return sum
}
```

#### 编码注意

不要忘记维护一个全局访问标记数组

#### 复杂度分析

* 时间复杂度O(mn)
* 空间复杂度O(mn)

