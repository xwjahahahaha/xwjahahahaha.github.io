---
title: 50.实现Pow
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-02-11 23:54:05
---

## 题目描述

[50. Pow(x, n)](https://leetcode-cn.com/problems/powx-n/)

难度中等

实现 [pow(*x*, *n*)](https://www.cplusplus.com/reference/valarray/pow/) ，即计算 `x` 的 `n` 次幂函数（即，`xn` ）。

<!-- more --> 

**示例 1：**

```
输入：x = 2.00000, n = 10
输出：1024.00000
```

**示例 2：**

```
输入：x = 2.10000, n = 3
输出：9.26100
```

**示例 3：**

```
输入：x = 2.00000, n = -2
输出：0.25000
解释：2-2 = 1/22 = 1/4 = 0.25
```

 

**提示：**

- `-100.0 < x < 100.0`
- `-231 <= n <= 231-1`
- `-104 <= xn <= 104`

通过次数253,561

提交次数670,623

## 解题思路及代码

```go
func myPow(x float64, n int) float64 {
    if x == 0.0 {
        return 0.0
    }
    if n == 0 {
        return 1.0
    }
    if n < 0 {
        return 1 / quickMul(x, -1*n)
    }
    return quickMul(x, n)
}

// 快速幂
func quickMul(x float64, n int) float64 {
    if n == 0 {
        return 1.0
    }
    y := quickMul(x, n/2)
    if n % 2 == 0 {
        return  y * y
    }else {
        return y * y * x
    }
}
```

