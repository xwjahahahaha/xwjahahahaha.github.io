---
title: 470.用Rand7实现Rand10
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-02-13 14:58:21
---

## 题目描述

[470. 用 Rand7() 实现 Rand10()](https://leetcode-cn.com/problems/implement-rand10-using-rand7/)

难度中等370

给定方法 `rand7` 可生成 `[1,7]` 范围内的均匀随机整数，试写一个方法 `rand10` 生成 `[1,10]` 范围内的均匀随机整数。

你只能调用 `rand7()` 且不能调用其他方法。请不要使用系统的 `Math.random()` 方法。

<!-- more -->

每个测试用例将有一个内部参数 `n`，即你实现的函数 `rand10()` 在测试时将被调用的次数。请注意，这不是传递给 `rand10()` 的参数。

**示例 1:**

```
输入: 1
输出: [2]
```

**示例 2:**

```
输入: 2
输出: [2,8]
```

**示例 3:**

```
输入: 3
输出: [3,8,10]
```

 

**提示:**

- `1 <= n <= 105`

 

**进阶:**

- `rand7()`调用次数的 [期望值](https://en.wikipedia.org/wiki/Expected_value) 是多少 ?
- 你能否尽量少调用 `rand7()` ?

## 解题思路及代码

```go
// 方法一：用两个rand7分别随机横纵坐标，取前40个, 保证每个数被取到的概率为4/49
func rand10() int {
    for {
        row := rand7()              // 横坐标
        col := rand7()              // 纵坐标
        val := (row-1)*7 + col      // 对应的值
        if val <= 40 {              // 只有在前40个范围内才可
            // 转换/映射到[1,10]
            ans := val % 10
            if ans == 0 {
                return 10
            }
            return ans
        } 
    }
}

// 方法二：[1, 6]奇偶概率为1/2，[1, 5]每个数的概率为1/5, 两者结合每个数的概率就都是1/10
func rand10() int { 
    for {
        nums1 := rand7()
        nums2 := rand7()
        if nums1 > 6 || nums2 > 5 {
            // 拒绝采样，限定在范围内
            continue
        }
        if nums1%2==0 {
            return nums2        // [1,5]: 1/2 * 1/5
        } else {
            return 5+nums2      // [6:10]: 1/2 * 1/5
        }
    }
}

// 优化方法：利用每次被拒绝的区间作为随机生成数，减少rand7的调用次数
func rand10() int { 
    for {
        // 7 * 7 = 49 取 40
        row := rand7()
        col := rand7()
        val := (row-1)*7 + col
        if val <= 40 {
            return 1 + (val-1)%10       // 解决求余得0 
        }

        // 9 * 7 = 63 取 60
        rand9 := val-40
        col = rand7()
        val = (rand9-1)*7 + col 
        if val <= 60 {
            return 1 + (val-1)%10
        }

        // 3 * 7 = 21 取 20
        rand3 := val-60
        col = rand7()
        val = (rand3-1)*7 + col
        if val <= 20 {
            return 1 + (val-1)%10
        }

        // val == 21 重新开始
    }
}
```

