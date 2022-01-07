---
title: 剑指Offer17.打印从1到最大的n位数
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-07-20 11:47:16
---

## 题目描述

[剑指 Offer 17. 打印从1到最大的n位数](https://leetcode-cn.com/problems/da-yin-cong-1dao-zui-da-de-nwei-shu-lcof/)

难度简单

输入数字 `n`，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数 999。

**示例 1:**

```
输入: n = 1
输出: [1,2,3,4,5,6,7,8,9]
```

 <!-- more -->

说明：

- 用返回一个整数列表来代替打印
- n 为正整数

## 解题思路及代码

本题乍一看非常简单，但是核心考察的是**大数溢出问题**

不论是int64还是其他语言，变量都有上限，在输入n很大的情况下就会导致溢出，所以大数溢出问题的解决：

* <font color='#e54d42'>字符数组表示数字，由低到高</font>

在编写代码时还需要注意两个细节：

1. 例如n=3， 从1～999怎样判断该字符数组到达999？

   使用循环逐位判断’9’复杂度为O(n)，不是很好的办法。当其再加一就会导致进位，判断其进位是否到达最高位的上一位即可（即判断再加1是否到1000），这样时间复杂度为O(1)

2. 初始化字符数组都为’0’，如果数字没有n位，那么前面会有多个无意义的0，对于输出来说不合适，所以在字符数组大数转为对应的数字时需要清除掉前面的0 （虽然Go的Atoi自动会清除）

```go
// 直观的思路
// 但是没有考虑大数溢出问题
func printNumbers(n int) []int {
    // 创建结果数组
    res := make([]int, 0)
    // 输出
    for i:=1; i<=int(math.Pow10(n)-1); i++ {
        res = append(res, i)
    }
    // 返回 
    return res
}
```


```go
// 解法二：字符串模拟构造大数
func printNumbers(n int) []int {
    res := make([]int, 0)
    // 初始化
    numString := make([]byte, n+1)    // 多设置一位是为了判断循环是否结束
    // 全部位初始化为0，即使达不到n位，前面也为0
    for i:=0; i<len(numString); i++ {
        numString[i] = byte('0')
    }
    for !Increment(&numString) {
        // 将输出存储
        // 排除前面的0, 找到第一个1
        vaildIndex := 1
        for i:=1; i<len(numString); i++ {
            if numString[i] != '0' {
                vaildIndex = i
                break
            }
        }
        num, _ := strconv.Atoi(string(numString[vaildIndex:]))
        res = append(res, num)
    }
    return res 
}

// 字符串数字 +1 函数
// 注意为了改变原字节数组，所以传递指针
func Increment(numString *[]byte) bool {
    var carry byte          // 进位
    carry = 0  
    n := len(*numString)-1     
    for i:=n; i>=0; i-- {       // 此循环为了进位
        if i == 0 {
            // 到达最高位即可让外层循环退出，最高位(数组中下标为1)的上一位(数组中下标为0)为1，产生进位，所以结束
            return true
        }
        // 取当前位置数
        num := (*numString)[i] - '0' 
        // +1 只在末位(数组中下标为n)
        if i == n {
            num += 1
        }
        // 加上进位
        num += carry
        // 计算新的进位
        if num >= 10 {
            carry = 1
            num -= 10
            (*numString)[i] = '0' + num     // 将当前位写上，进位还有继续循环计算 
        }else {
            // 变回字符串
            (*numString)[i] = '0' + num     // 将当前位写上，已无进位，结束循环
            break
        }
    }
    // 一次+1 计算完毕，无溢出返回falsefalse，让外层循环继续写入
    return false
}
```

对于使用字符串表示大数，本题的大数每一位都是’0’～’9’字符的排列，所以可以看作全排列来减少代码量

全排列使用递归会更加的好写：

```go
// 大数字符串的递归全排列
// 每一位都是字符0～1的全排列
func printNumbers(n int) []int {
    res := make([]int, 0)
    numString := make([]byte, n)
    // 递归函数
    var recursive func(int) 
    recursive = func(index int) {
        // 递归结束条件
        if index == n {
            // 记录当前值（除去开头无效0）
            vaildStart := 0
            for i:=0; i<n; i++ {
                if numString[i] != '0' {
                    vaildStart = i
                    break
                }
            }
            num, _ := strconv.Atoi(string(numString[vaildStart:]))
            res = append(res, num)
            return
        }
        for i:=0; i<=9; i++ {
            // 存储当前下标位置字符
            numString[index] = '0' + byte(i)
            // 向下一位递归
            recursive(index+1)
        }
    }
    // 调用函数
    recursive(0)
    return res[1:]  // 此处去除掉最开始的0
}
```

> <font color='#39b54a'>本题直接用数字遍历到n位在时间上肯定会比字符串大数来的快，但是安全性上是有所缺陷的并且对于项目来说可能是致命的，具体场景要具体适用</font>

