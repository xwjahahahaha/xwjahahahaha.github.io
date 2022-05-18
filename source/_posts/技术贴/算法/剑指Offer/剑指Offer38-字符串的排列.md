---
title: 剑指Offer38.字符串的排列
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-04-29 16:26:37
---

## 题目描述

#### [剑指 Offer 38. 字符串的排列](https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/)

难度中等

输入一个字符串，打印出该字符串中字符的所有排列。

你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。

<!-- more -->

**示例:**

```
输入：s = "abc"
输出：["abc","acb","bac","bca","cab","cba"]
```

**限制：**

```
1 <= s 的长度 <= 8
```

## 解题思路及代码

方法一：hash表去重

```go
func permutation(s string) []string {
    ans, n := []string{}, len(s)
    visitedMap, sMap := make(map[int]bool), make(map[string]bool)
    var dfs func(path string) 
    dfs = func(path string) {
        if len(path) == n {
            // 去重复
            if _, has := sMap[path]; !has {
                ans = append(ans, path)
                sMap[path] = true   
            }
            return
        }
        for i:=0; i<n; i++ {
            if visitedMap[i] {
                continue
            }
            path += string(s[i])
            visitedMap[i] = true
            dfs(path)
            // 回溯
            path = path[:len(path)-1]
            visitedMap[i] = false
        }
    }
    dfs("")
    return ans
}
```

方法二：排序去重

```go
// 先排序，一个位置只用一个字母
func permutation(s string) []string {
    n := len(s)
    visitedMap := make(map[int]bool)
    ans, sBytes := []string{}, []byte{}
    for i:=0; i<n; i++ {
        sBytes = append(sBytes, s[i])
    }
    // 排序
    sort.SliceStable(sBytes, func(i, j int) bool {
        return sBytes[i] < sBytes[j]
    })
    var dfs func(path string) 
    dfs = func(path string) {
        if len(path) == n {
            ans = append(ans, path)
            return
        }
        // 确定当前第i个位置
        for i:=0; i<n; i++ {
            // 多加一个对于相同字母集的过滤(只取第一个)
            // 注意!visitedMap[i-1]这个条件
            if i > 0 && !visitedMap[i-1] && sBytes[i] == sBytes[i-1] {
                continue
            }
            if visitedMap[i] {
                continue
            }
            path += string(sBytes[i])
            visitedMap[i] = true
            dfs(path)
            // 回溯
            path = path[:len(path)-1]
            visitedMap[i] = false
        }
    }
    dfs("")
    return ans
}
```

`!visitedMap[i-1]` 添加此判断的原因在于：

以`aab`为例：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/IMG_490C62A59B14-1.jpeg" alt="IMG_490C62A59B14-1" style="zoom:33%;" />

* 1剪枝的原因：先选择了第二个a，第一个a此时因为已经回溯所以`!visitedMap[i-1]`为true且`sBytes[i] == sBytes[i-1]`，所以剪枝
* 2剪枝的原因：与1相同，此时选了第二个a和b，第一个a未选，所以同样满足条件剪枝
* 3剪枝的原因：因为一个a没选，所以满足条件，剪枝

总结：只要是第一个a没有选择，那么就不能选择第二个a（剪枝），所以需要这样的一个条件
