---
title: 剑指Offer14-I.剪绳子
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-07-13 17:02:06
---

## 题目描述

[剑指 Offer 14- I. 剪绳子](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/)

难度中等

给你一根长度为 `n` 的绳子，请把绳子剪成整数长度的 `m` 段（m、n都是整数，n>1并且m>1），每段绳子的长度记为 `k[0],k[1]...k[m-1]`。请问 `k[0]*k[1]*...*k[m-1]` 可能的最大乘积是多少？例如，当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到的最大乘积是18。

<!-- more -->

**示例 1：**

```
输入: 2
输出: 1
解释: 2 = 1 + 1, 1 × 1 = 1
```

**示例 2:**

```
输入: 10
输出: 36
解释: 10 = 3 + 3 + 4, 3 × 3 × 4 = 36
```

**提示：**

- `2 <= n <= 58`

## 解题思路及代码

### 方法一： 动态规划

时间复杂度O($n^2$), 空间复杂度O(n)

<font color='#e54d42'>**从上往下分析，从下往上编写**</font> (避免重复计算)

* 对于dp[i]长度的绳子，可从k位置开始剪开，则k的范围为[1, i)
* 在第k个位置剪开，[k, i]长度的绳子可以剪也可以不剪，选择大者。则状态转移方程为: $dp[i] = max(k*dp[i-k], k*(i-k))$
* 对于所有的k位置，取最大为当前dp[i]的最大乘积，所以状态转移方程变为:$dp[i] = max(dp[i], max(k*dp[i-k], k*(i-k)))$
* 优化: 对于长度为i的绳子，剪为长度k与(i-k) 和 剪为长度(i-k)和k是一样的，所以第k个位置剪开只需要考虑到i/2不用到i

```go
// 动态规划 时间复杂度O(n2), 空间复杂度O(n)
// 构建动态规划方程：
// dp[i] = max{dp[i], max{k*dp[i-k], k*(i-k)}}
func cuttingRope(n int) int {
    // 创建动态规划数组
    dp := make([]int, n+1)
    // 初始化情况
    // 长度为2最大乘积为1
    dp[1], dp[2] = 1, 1           
    // 从下往上构建,避免重复计算
    for i:=3; i<=n; i++ {
        // 遍历切的位置k
        max := 0
        // 此循环优化：不用到达i，切到中间后面都是一样的
        for k:=1; k<=i/2; k++ {
            // [k, i]部分裁剪
            clip := k * dp[i-k]
            // [k, i]部分不裁剪
            no_clip := k * (i-k)
            // 选择最大的
            if clip > no_clip {
                dp[i] = clip
            } else {
                dp[i] = no_clip
            }
            // 找出所有裁剪位置的最大值
            if dp[i] > max {
                max = dp[i]
            } 
        }
        dp[i] = max
    }
    return dp[n]
}
```

> 复杂度优化思路加一：
>
> * 考虑遍历过程前后是否有重复，有的话可以只遍历一半，甚至开根

### 方法二 贪心算法

时间复杂度O（n）

尽可能把绳子分成长度为3的小段，这样乘积最大，当最后为4时分为两个2最大

```go
// 贪心算法 时间复杂度O（n）
// 尽可能把绳子分成长度为3的小段，这样乘积最大
func cuttingRope(n int) int {
    // 直接返回一些特殊情况
    if n == 1 || n == 2{
        return 1
    } else if n == 3 {
        return 2
    } 
    // 计算3的次数
    timesOf3 := n / 3
    // 如果最后一次为4，那么就改为2*2
    if n - timesOf3 * 3 == 1 {
        // 现将最后一次剪次数扣除
        timesOf3 -= 1       
    }
    // 计算余下2的次数，要么为一次(余2)，要么为2次(余4)
    timesOf2 := (n - timesOf3 * 3) / 2

    return int(math.Pow(float64(3), float64(timesOf3))) * int(math.Pow(float64(2), float64(timesOf2)))

}
```

