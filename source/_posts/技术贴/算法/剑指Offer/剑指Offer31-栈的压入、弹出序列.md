---
title: 剑指Offer31.栈的压入、弹出序列
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-23 10:47:47
---

## 题目描述

[剑指 Offer 31. 栈的压入、弹出序列](https://leetcode-cn.com/problems/zhan-de-ya-ru-dan-chu-xu-lie-lcof/)

难度中等

输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等。例如，序列 {1,2,3,4,5} 是某栈的压栈序列，序列 {4,5,3,2,1} 是该压栈序列对应的一个弹出序列，但 {4,3,5,1,2} 就不可能是该压栈序列的弹出序列。

 

**示例 1：**

```
输入：pushed = [1,2,3,4,5], popped = [4,5,3,2,1]
输出：true
解释：我们可以按以下顺序执行：
push(1), push(2), push(3), push(4), pop() -> 4,
push(5), pop() -> 5, pop() -> 3, pop() -> 2, pop() -> 1
```

**示例 2：**

```
输入：pushed = [1,2,3,4,5], popped = [4,3,5,1,2]
输出：false
解释：1 不能在 2 之前弹出。
```

 <!-- more -->

**提示：**

1. `0 <= pushed.length == popped.length <= 1000`
2. `0 <= pushed[i], popped[i] < 1000`
3. `pushed` 是 `popped` 的排列。

注意：本题与主站 946 题相同：https://leetcode-cn.com/problems/validate-stack-sequences/



## 解题思路及代码

```go
func validateStackSequences(pushed []int, popped []int) bool {
    // 根据出栈顺序确定压栈顺序
    stack := []int{}
    i, j := 0, 0
    for j<len(pushed) && i<len(popped) {  // 循环压栈序列
        if pushed[j] != popped[i] {
            if len(stack) > 0 && popped[i] == stack[len(stack)-1] {
                // 如果当前辅助栈栈顶元素与当前考察的出栈序列中元素相同，则出栈匹配
                stack = stack[:len(stack)-1]
                i++
            }else {
                // 如果当前辅助栈为空
                // 或者如果当前辅助栈中栈顶元素与出栈序列数也不同，则压入辅助栈
                stack = append(stack, pushed[j])
                j++
            }
        }else {
            // 如果出栈序列元素与压栈序列元素相等，则指针同时向前一步
            i++
            j++
        }
    }
    if j == len(pushed) && i < len(popped) {
        // 压栈序列结束，出栈序列未结束，只能出辅助栈元素，一一比对
        for len(stack) > 0 {
            if stack[len(stack)-1] != popped[i] {
                return false
            }
            stack = stack[:len(stack)-1]
            i++
        }
    } 
    return true
}
```

