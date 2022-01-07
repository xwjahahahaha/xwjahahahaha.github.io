---
title: 剑指Offer06.从尾到头打印链表
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-08 10:08:19
---

## 题目描述

[剑指 Offer 06. 从尾到头打印链表](https://leetcode-cn.com/problems/cong-wei-dao-tou-da-yin-lian-biao-lcof/)

难度简单155

输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

**示例 1：**

```
输入：head = [1,3,2]
输出：[2,3,1]
```

**限制：**

```
0 <= 链表长度 <= 10000
```


<!-- more -->

## 解题思路及代码

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 直接的思路: 栈
// 时间复杂度O(N) 空间复杂度O(N)
func reversePrint(head *ListNode) []int {
    stack := []*ListNode{}
    p := head
    // 入栈
    for p != nil {
        stack = append(stack, p)
        p = p.Next
    }
    // 出栈
    ans := []int{}
    for len(stack) > 0 {
        n := len(stack)
        ans = append(ans, stack[n-1].Val)
        stack = stack[:n-1]
    }
    return ans
}

// 递归的本质就是栈, 采用递归的思路同解
func reversePrint(head *ListNode) []int {
    ans := []int{}
    var recursive func(*ListNode)
    recursive = func(node *ListNode) {
        if node == nil {
            return
        }
        recursive(node.Next)
        ans = append(ans, node.Val)
    }
    recursive(head)
    return ans
}
```

