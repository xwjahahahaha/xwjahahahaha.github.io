---
title: 剑指Offer05.替换空格
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-08 09:53:11
---

## 题目描述

[剑指 Offer 05. 替换空格](https://leetcode-cn.com/problems/ti-huan-kong-ge-lcof/)

难度简单122

请实现一个函数，把字符串 `s` 中的每个空格替换成"%20"。

 **示例 1：**

```
输入：s = "We are happy."
输出："We%20are%20happy."
```

**限制：**

```
0 <= s 的长度 <= 10000
```


<!-- more -->

## 解题思路及代码

```go
// 时间复杂度O(N), 空间复杂度O(N)
// 缺点: 使用额外空间
func replaceSpace(s string) string {
    n := len(s) 
    if n == 0 {
        return s
    }
    var ans []byte
    for i:=0; i<n; i++ {
        if s[i] == ' ' {
            ans = append(ans, []byte("%20")...)
        }else {
            ans = append(ans, s[i])
        }    
    }
    return string(ans)
}

// 从后往前,双指针
// 时间复杂度O(N), 空间复杂度O(N)
func replaceSpace(s string) string {
    n := len(s)
    if n == 0 {
        return s
    }
    // 统计空格个数
    count := 0
    for _, v := range s {
        if v == ' ' {
            count ++
        }
    }
    ans := make([]byte, 2*count+n)
    p1, p2 := n-1, 2*count+n-1
    for p1 >= 0 {
        if s[p1] == ' ' {
            ans[p2] = '0'
            p2 --
            ans[p2] = '2'
            p2 --
            ans[p2] = '%'
        }else {
            ans[p2] = s[p1]
        }
        p1 --
        p2 -- 
    }
    return string(ans)
}
```

