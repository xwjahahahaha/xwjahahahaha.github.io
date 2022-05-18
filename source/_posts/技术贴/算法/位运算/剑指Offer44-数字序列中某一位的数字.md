---
title: 剑指Offer44.数字序列中某一位的数字
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-03 22:11:46
---

## 题目描述

[剑指 Offer 44. 数字序列中某一位的数字](https://leetcode-cn.com/problems/shu-zi-xu-lie-zhong-mou-yi-wei-de-shu-zi-lcof/)

难度中等

数字以`0123456789101112131415…`的格式序列化到一个字符序列中。在这个序列中，第5位（从下标0开始计数）是5，第13位是1，第19位是4，等等。

请写一个函数，求任意第n位对应的数字。

 <!-- more -->

**示例 1：**

```
输入：n = 3
输出：3
```

**示例 2：**

```
输入：n = 11
输出：0
```

**限制：**

- `0 <= n < 2^31`

注意：本题与主站 400 题相同：https://leetcode-cn.com/problems/nth-digit/

## 解题思路及代码

```go
// 注意：此题数列从0开始，计数也从0开始，所以不用考虑0，直接认为从1开始计数也从1开始
func findNthDigit(n int) int {
    if n == 0 {                         // 特殊判断掉0
        return 0
    }
    // 1. 确定n是几位数
    d, count := 0, 0          
    for n>count {                       // d表示几位数
        // 累加当前
        d++
        count += 9*pow(10, d-1)*d       // 共9*pow(10, d-1)个数，每个数d位
    } 
    count -= 9*pow(10, d-1)*d
    // 计算多出的位数
    overFlow := n-count
    // 2. 确定是哪个数
    head := pow(10, d-1)                // d位数的第一个数，例如2位数的第一个数就是10
    divRes := upRound(overFlow, d)      // 注意：是上取整数
    modRes := overFlow%d
    num := head + divRes -1
    // fmt.Println(num, modRes)
    numStr := strconv.Itoa(num)
    if modRes == 0 {                    // 注意点：如果是0则代表是这个数的最后一位
        return int(numStr[len(numStr)-1]-'0')
    }
    return int(numStr[modRes-1]-'0')
}

func pow(a, b int) int {
    if a < 0 || b < 0 {
        return -1
    }
    if a == 0 {
        return 0
    }
    if b == 0 {
        return 1
    }
    half := pow(a, b/2)
    if b % 2 == 0 {
        return half*half
    }
    return half*half*a
}

// 上取整
func upRound(a, b int) int {
    c := a / b 
    if b*c < a {
        return c+1
    }
    return c
}
```

