---
title: NC1.大数加法
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-12-01 20:05:25
---

## 题目描述

[NC1 大数加法](https://www.nowcoder.com/practice/11ae12e8c6fe48f883cad618c2e81475?tpId=188&&tqId=38569&rp=1&ru=/ta/job-code-high-week&qru=/ta/job-code-high-week/question-ranking)

中等 通过率：42.07% 时间限制：1秒 空间限制：256M

知识点[字符串](https://www.nowcoder.com/ta/job-code-high-week?tag=579)[模拟](https://www.nowcoder.com/ta/job-code-high-week?tag=595)

<!-- more -->

**描述**

以字符串的形式读入两个数字，编写一个函数计算它们的和，以字符串形式返回。 

数据范围：len(s),len(t) ≤ 100000，字符串仅由'0'~‘9’构成 

要求：时间复杂度 O(n)

**示例1**

输入：

```
"1","99"
```

返回值：

```
"100"
```

说明：

```
1+99=100     
```

**示例2**

输入：

```
"114514",""
```

返回值：

```
"114514"
```

## 解题思路及代码

```go
package main

// import "fmt"

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 * 计算两个数之和
 * @param s string字符串 表示第一个整数
 * @param t string字符串 表示第二个整数
 * @return string字符串
*/
func solve( s string ,  t string ) string {
    // write code here
    res, carry := "", 0
    for p1, p2 := len(s)-1, len(t)-1; p1 >= 0 || p2 >= 0; p1, p2 = p1-1, p2-1 {
        // 当有一个数的位数较短时，默认为0
        nums1, nums2 := 0, 0
        if p1 >= 0 { nums1 = int(s[p1]-'0') } 
        if p2 >= 0 { nums2 = int(t[p2]-'0') } 
        sum := nums1 + nums2 + carry        // 加上进位
        res = string(byte(sum % 10 + '0')) + res
        carry = sum / 10
    }
    // 如果最高位还有进位，则加上最后的结果
    if carry != 0 {
        res = string(byte(carry + '0')) + res
    }
    return res
}
```

二刷：

```go
package main

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 * 计算两个数之和
 * @param s string字符串 表示第一个整数
 * @param t string字符串 表示第二个整数
 * @return string字符串
*/
func solve( s string ,  t string ) string {
    m, n := len(s), len(t)
    p1, p2 := m-1, n-1
    res, carry := "", 0
    
    for p1 >= 0 || p2 >= 0 {
        num1, num2 := 0, 0
        if p1 >= 0 {
            num1 = int(s[p1]-'0')
        }
        if p2 >= 0 {
            num2 = int(t[p2]-'0')
        }
        
        sum := num1 + num2 + carry
        res = string(sum%10+'0') + res
        carry = sum/10
        
        p1--; p2--
    }
    
    // 处理carry剩余
    if carry != 0 {
        res = string(carry+'0') + res
    }
    
    return res
}
```

