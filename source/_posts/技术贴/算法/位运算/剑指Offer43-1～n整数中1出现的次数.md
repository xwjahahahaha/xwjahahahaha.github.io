---
title: 剑指Offer43.1～n整数中1出现的次数
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-03 15:01:50
---

## 题目描述

[剑指 Offer 43. 1～n 整数中 1 出现的次数](https://leetcode-cn.com/problems/1nzheng-shu-zhong-1chu-xian-de-ci-shu-lcof/)

难度困难

输入一个整数 `n` ，求1～n这n个整数的十进制表示中1出现的次数。

例如，输入12，1～12这些整数中包含1 的数字有1、10、11和12，1一共出现了5次。

 <!-- more -->

**示例 1：**

```
输入：n = 12
输出：5
```

**示例 2：**

```
输入：n = 13
输出：6
```

 

**限制：**

- `1 <= n < 2^31`

注意：本题与主站 233 题相同：https://leetcode-cn.com/problems/number-of-digit-one/

## 解题思路及代码

解题详细思路见：https://leetcode-cn.com/problems/1nzheng-shu-zhong-1chu-xian-de-ci-shu-lcof/solution/dong-hua-mo-ni-wo-tai-xi-huan-zhe-ge-ti-vxzwc/

```go
// 当前位：考虑当前为1的位
// 高位：大于当前位的所有位
// 低位：小于当前位的所有位
// 例如：1024，当前位在十位(num就是10，类推百位就是100)，当前位数字为2，高位为"10"，低位为"4"
// 核心思想：固定某一位为1，求其次数，所有位为1的次数之和就是结果
func countDigitOne(n int) int {
    count := 0
    // 遍历当前位(1, 10, 100, ...)
    for num:=1; num<=n; num*=10 {
        // 判断当前位的数字类型
        switch n / num % 10 {
        // 当前位的三种类型
        case 0:
            // ==0
            count += (n/num/10) * num                         // 次数=高位*num 
        case 1:
            // ==1
            count += (n/num/10) * num + (n % num) + 1         // 次数=高位*num+低位+1
        default:
            // >1
            count += (n/num/10) * num + num                   // 次数=高位*num+num
        }
    }
    return count
}
```

* 时间复杂度`O(logn)`：因为循环的次数是数字n的位数
* 空间复杂度`O(1)`
