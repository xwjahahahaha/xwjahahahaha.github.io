---
title: 剑指Offer16.数值的整数次方
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-07-19 11:05:02
---

## 题目描述

[剑指 Offer 16. 数值的整数次方](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/)

难度中等

实现 [pow(*x*, *n*)](https://www.cplusplus.com/reference/valarray/pow/) ，即计算 x 的 n 次幂函数（即，xn）。不得使用库函数，同时不需要考虑大数问题。

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



## 解题思路及代码

此题细节非常多，考察的不仅仅是简单的求整数次方，要着重考虑**边界情况与优化**

细节注意点：

1. 输入的指数n是否可能为负数，负数要怎样处理？
2. 输入的底数x是否可能为0，n为负数，那么就会导致分母为0的情况，所以对于x为0统一的处理方式是返回0（$0^0$在数学上没有意义）
3. 浮点数的等值判断是否可以直接用等号？显然是不合适的
4. 求数值的整数次方时间上是否可以优化？
5. 除2以及求余函数是否可以优化？

求整数倍的优化算法是：
$$
a^n = \begin{cases}a^{n/2} * a^{n/2} & n为偶数 \\ a^{(n-1)/2} * a^{(n-1)/2} * a & n为奇数 \end{cases}
$$
详情可见下方代码：

```go
const MIN = 0.00000000000001        // 比较的精度

// 比较两个浮点数是否相等
func IsEqual(f1, f2 float64) bool {
    if f1 > f2 {
        return math.Dim(f1, f2) < MIN
    }else {
        return math.Dim(f2, f1) < MIN
    }
}

// 数值整数次方计算函数
// 递归
func MyPow(x float64, n uint) float64 {
    if n == 0 {
        return 1.0
    }else if n == 1 {
        return x
    }
    // 右移代替除二
    res := MyPow(x, n>>1)
    res = res * res
    // 判断奇偶，奇数再乘本身一次
    // 位与运算代替求余
    if n & 0x1 == 1 {
        res *= x
    }             
    return res
}

func myPow(x float64, n int) float64 {
    if n == 0 {
        return 1.0
    }else if IsEqual(x, 0.0) {       // 如果底数为0，则无意义直接返回0
        return 0.0
    }else if n < 0 {                // 如果底数不为0，但指数为负数，那么处理底数与指数
        x = 1.0 / x
        n *= -1
    }
    // 求整数倍的优化算法
    return MyPow(x, uint(n))
}
```

二刷

```go
func myPow(x float64, n int) float64 {
    // 特殊情况, 0^0=1
    if n == 0 {
        return 1
    }
    if x == 0 {
        return 0
    }

    if n < 0 {
        return 1 / pow(x, -n)
    }
    
    return pow(x, n)
}

// 快速幂（二分法）
func pow(x float64, n int) float64 {
    if n == 1 {
        return x
    }
    if n == 0 {
        return 1
    }

    half := pow(x, n/2)
    if n % 2 == 0 {
        return half * half
    }else {
        return half * half * x
    }
}
```

