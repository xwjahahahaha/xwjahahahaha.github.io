---
title: 剑指Offer20.表示数值的字符串
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-08-03 13:40:04
---

## 题目描述

[剑指 Offer 20. 表示数值的字符串](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/)

难度中等

请实现一个函数用来判断字符串是否表示**数值**（包括整数和小数）。

<!-- more -->

**数值**（按顺序）可以分成以下几个部分：

1. 若干空格
2. 一个 **小数** 或者 **整数**
3. （可选）一个 `'e'` 或 `'E'` ，后面跟着一个 **整数**
4. 若干空格

**小数**（按顺序）可以分成以下几个部分：

1. （可选）一个符号字符（`'+'` 或 `'-'`）
2. 下述格式之一：
   1. 至少一位数字，后面跟着一个点 `'.'`
   2. 至少一位数字，后面跟着一个点 `'.'` ，后面再跟着至少一位数字
   3. 一个点 `'.'` ，后面跟着至少一位数字

**整数**（按顺序）可以分成以下几个部分：

1. （可选）一个符号字符（`'+'` 或 `'-'`）
2. 至少一位数字

部分**数值**列举如下：

- `["+100", "5e2", "-123", "3.1416", "-1E-16", "0123"]`

部分**非数值**列举如下：

- `["12e", "1a3.14", "1.2.3", "+-5", "12e+5.4"]`

 

**示例 1：**

```
输入：s = "0"
输出：true
```

**示例 2：**

```
输入：s = "e"
输出：false
```

**示例 3：**

```
输入：s = "."
输出：false
```

**示例 4：**

```
输入：s = "    .1  "
输出：true
```

 

**提示：**

- `1 <= s.length <= 20`
- `s` 仅含英文字母（大写和小写），数字（`0-9`），加号 `'+'` ，减号 `'-'`，空格 `' '` 或者点 `'.'` 。




## 解题思路及代码

分析出数值的匹配模式，写出代码

先看一下书上的分析

![image-20210803134344324](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210803134344324.png)

分析完之后编写代码时需要注意的是对于多种逻辑情况“&&”和“||”的巧妙使用

```go
// 匹配模式： A[.[B]][e|EC] || .B[e|EC]
// 其中A为整数部分，B为小数部分，C为指数部分
func isNumber(s string) bool {
    var res bool
    // 去除收尾空白
    s = strings.Trim(s, " ")
    n := len(s)
    // 扫描整数A
    index, resA := scanInteger(s, 0)
    res = resA

    // 用两个if表示上面的匹配模式的关系

    // 判断是否有小数部分
    if index < n && s[index] == '.' {
        index ++ 
        // 扫描小数部分B
        // B只能是无符号整型
        indexB, resB := scanUnsignInteger(s, index)     
        index = indexB
        // 用||的原因:
        // resA = false 表示：整数A部分可能没有，但也可能是正确的, 例如.35(省略0)
        // resB = false 表示：没有小数部分也是可以的，例如9.(省略小数点0)
        // 只有两者同时为false，那么表示一定是错误的，例如.e3 
        res = resA || resB            // 细节：使用或
    }
    
    // 判断指数部分
    if index < n && (s[index] == 'e' || s[index] == 'E') {
        index ++ 
        // 扫描指数部分C
        indexC, resC := scanInteger(s, index) 
        index = indexC
        // 用&&的原因
        // res = false 表示整数小数部分都不存在即前面没数字不是数值，例如.e1、e3
        // res = true 表示前面A、B部分都没问题，看C的结果
        // resC = false 表示e｜E后面没数字，则是错误的, 例如12e
        // 都为true则正确
        res = resC && res             // 细节：与
    }

    // 返回结果
    return res && index == n
}


// 扫描有符号数
func scanInteger(s string, i int) (int, bool) {
    // 扫描符号
    if i < len(s) && (s[i] == '+' || s[i] == '-') {
        i++
    }
    return scanUnsignInteger(s, i)
}

// 扫描无符号数
func scanUnsignInteger(s string, i int) (int, bool) {
    j := i
    for j < len(s) && s[j] >= '0' && s[j] <= '9' {
        j++
    }
    // 若存在0～9的字符串时，返回true
    return j, j > i 
}
```

