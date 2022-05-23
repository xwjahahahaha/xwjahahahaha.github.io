---
title: 剑指Offer48.最长不含重复字符的子字符串
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-05-22 22:58:39
---

## 题目描述

[剑指 Offer 48. 最长不含重复字符的子字符串](https://leetcode.cn/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/)

难度中等

请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。

<!-- more --> 

**示例 1:**

```
输入: "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

**示例 2:**

```
输入: "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

**示例 3:**

```
输入: "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

提示：

- `s.length <= 40000`

注意：本题与主站 3 题相同：https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/

## 解题思路及代码

```go
// 滑动窗口
func lengthOfLongestSubstring(s string) int {
    count := make(map[byte]bool)
    max := 0
    n, l, r := len(s), 0, 0
    for r < n {
        if !count[s[r]] {
            count[s[r]] = true
            // 计算最大长度
            if r-l+1 > max {
                max = r-l+1
            }
            r++
            continue
        } 
        // 滑动左窗口
        for l <= r && count[s[r]] {
            delete(count, s[l])
            l++
        }
    }
    return max
}
```

