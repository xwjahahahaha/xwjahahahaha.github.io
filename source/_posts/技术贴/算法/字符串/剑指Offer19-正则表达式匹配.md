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

牛客二刷：

```go
package main

/**
 * 代码中的类名、方法名、参数名已经指定，请勿修改，直接返回方法规定的值即可
 *
 * 
 * @param str string字符串 
 * @param pattern string字符串 
 * @return bool布尔型
*/
func match( str string ,  pattern string ) bool {
    n, m := len(str), len(pattern)
    if n == 0 && m == 0 {        // 匹配完毕
        return true
    }else if m == 0 {
        return false            // 当匹配字符串为空串但str不是空串时一定返回false
    }else if n == 0 {
        // 当str为空串时，只有这一种情况才会匹配，否则直接返回false
        if m == 2 && pattern[1] == '*' {
            return true
        }
        return false
    }
    // 比较两个字符
    // 判断模式字符串的下一个字符是*号
    if m > 1 && pattern[1] == '*' {
        // 是*号，则分类讨论
        // 判断第一个字符是否相同
        if str[0] == pattern[0] || pattern[0] == '.' {
            // 相同
            // 1. *只匹配一次的情况
            if match(str[1:], pattern[2:]) {
                return true
            }
            // 2. *多次匹配
            if match(str[1:], pattern) {
                return true
            }
            // 3. *不匹配，即使相同也不匹配,也有这种情况
            if match(str, pattern[2:]) {
                return true
            }
        }else {
            // 4. 因为不同，所以只能不匹配
            return match(str, pattern[2:])
        }
    }else {
        // 不是则直接匹配
        if pattern[0] == '.' || pattern[0] == str[0] {
            return match(str[1:], pattern[1:])
        }
    }
    return false
}
```

三刷：

```go
func isMatch(s string, p string) bool {
    return help(s, p, 0, 0)
}

func help(s, p string, i, j int) bool {
    m, n := len(s), len(p)
    if i == m && j ==n {
        return true
    }

    if i < m && j >= n {
        return false
    }

    // 判断第二个字符
    if j+1 < n && p[j+1] == '*' {
        if i<m && (s[i] == p[j] || p[j] == '.') {
            // 第一个字符匹配
            // 匹配0次、1次、多次
            return help(s, p, i, j+2) || help(s, p, i+1, j+2) || help(s, p, i+1, j)
        }else {
            // 匹配0次
            return help(s, p, i, j+2)
        }
    }else {
        // 第二个字符不是*
        // 看第一个字符是否匹配
        if i<m && (s[i] == p[j] || p[j] == '.') {
            return help(s, p, i+1, j+1)
        }else {
            // 第一个字符也不匹配直接返回false
            return false
        }
    }
}
```

