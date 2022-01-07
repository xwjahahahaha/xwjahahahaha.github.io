---
title: 剑指Offer19.正则表达式匹配
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-08-02 09:55:04
---

## 题目描述

[剑指 Offer 19. 正则表达式匹配](https://leetcode-cn.com/problems/zheng-ze-biao-da-shi-pi-pei-lcof/)

难度**困难**

请实现一个函数用来匹配包含`'. '`和`'*'`的正则表达式。模式中的字符`'.'`表示任意一个字符，而`'*'`表示它前面的字符可以出现任意次（含0次）。在本题中，匹配是指字符串的所有字符匹配整个模式。例如，字符串`"aaa"`与模式`"a.a"`和`"ab*ac*a"`匹配，但与`"aa.a"`和`"ab*a"`均不匹配。

<!-- more -->

**示例 1:**

```
输入:
s = "aa"
p = "a"
输出: false
解释: "a" 无法匹配 "aa" 整个字符串。
```

**示例 2:**

```
输入:
s = "aa"
p = "a*"
输出: true
解释: 因为 '*' 代表可以匹配零个或多个前面的那一个元素, 在这里前面的元素就是 'a'。因此，字符串 "aa" 可被视为 'a' 重复了一次。
```

**示例 3:**

```
输入:
s = "ab"
p = ".*"
输出: true
解释: ".*" 表示可匹配零个或多个（'*'）任意字符（'.'）。
```

**示例 4:**

```
输入:
s = "aab"
p = "c*a*b"
输出: true
解释: 因为 '*' 表示零个或多个，这里 'c' 为 0 个, 'a' 被重复一次。因此可以匹配字符串 "aab"。
```

**示例 5:**

```
输入:
s = "mississippi"
p = "mis*is*p*."
输出: false
```

- `s` 可能为空，且只包含从 `a-z` 的小写字母。
- `p` 可能为空，且只包含从 `a-z` 的小写字母以及字符 `.` 和 `*`，无连续的 `'*'`。

注意：本题与主站 10 题相同：https://leetcode-cn.com/problems/regular-expression-matching/

## 解题思路及代码

此题目与计算理论基础中的**NFA非确定有限状态机**是类似的

先看一下书上的解释：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802101539083.png" alt="image-20210802101539083" style="zoom:80%;" />



<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802101253644.png" alt="image-20210802101253644" style="zoom:87%;" />

* 对于模式中的“.”较为简单，其匹配任意字符

* 对于模式中的“*”则需要注意多种情况, 具体的逻辑思路如下图：

![image-20210802100813377](http://xwjpics.gumptlu.work/qinniu_uPic/image-20210802100813377.png)

结合代码理解如下：

```go
func isMatch(s string, p string) bool {
    return matchCore(s, p, 0, 0)
}

// 递归匹配函数
// i, j分别代表标
func matchCore(s, p string, i, j int) bool {
    m, n := len(s), len(p)
    // 递归结束情况
    if i == m && j == n {
        // 同时匹配结束，那么返回true
        return true
    }
    if i < m && j >= n {    // 细节1
        // 如果字符串没匹配完且模式匹配结束那么总体一定是匹配失败
        // 注意：这里不能以判断字符串是否到达末尾来判断总体结果，因为有情况是即使字符串到达末尾，模式也可以为空串匹配
        return false
    }
    // 如果第二个字符是*
    if j+1 < n && p[j+1] == '*' {
        // 判断第一个位置是否匹配
        if i < m && (s[i] == p[j] || p[j] == '.') { 
            // 第一个位置匹配, 则分为三种情况
            // 1. *使用一次 => 模式跳过2个字符
            // 2. *使用多次 => 模式不动（下次再用）
            // 3. *不使用即空串0次 => 模式跳过，字符串不动
            return matchCore(s, p, i+1, j+2) || matchCore(s, p, i+1, j) || matchCore(s, p, i, j+2)
        }else {
            // 第一个位置不匹配，则直接让*为空串即0
            return matchCore(s, p, i, j+2)  // i不变，j直接跳过两个字符
        }
    }else {
        // 第二个字符不是*，则只需要匹配当前字符
        if i < m && (s[i] == p[j] || p[j] == '.') {
            return matchCore(s, p, i+1, j+1)    // 字符串与模式同时+1
        }
    }
    return false
}
```

总结：

* <font color='#e54d42'>逻辑产生分支即多种情况可以使用多个递归并用逻辑表达式连接的结构编写代码</font>

