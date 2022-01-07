---
title: 剑指Offer22.链表中倒数第k个节点
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-13 14:36:58
---

## 题目描述

[剑指 Offer 22. 链表中倒数第k个节点](https://leetcode-cn.com/problems/lian-biao-zhong-dao-shu-di-kge-jie-dian-lcof/)

难度简单

输入一个链表，输出该链表中倒数第k个节点。为了符合大多数人的习惯，本题从1开始计数，即链表的尾节点是倒数第1个节点。

例如，一个链表有 `6` 个节点，从头节点开始，它们的值依次是 `1、2、3、4、5、6`。这个链表的倒数第 `3` 个节点是值为 `4` 的节点。

<!-- more --> 

**示例：**

```
给定一个链表: 1->2->3->4->5, 和 k = 2.

返回链表 4->5.
```

## 解题思路及代码

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 遍历两次: 找正向第n-k+1个, 走n-k步
// 时间复杂度O(N),空间复杂度O(1)
func getKthFromEnd(head *ListNode, k int) *ListNode {
    p, n := head, 0
    for p != nil {
        n ++
        p = p.Next
    }
    p = head
    for i:=1; i<=n-k; i++ {
        p = p.Next
    }
    return p
}

// 遍历一次的思路：空间换时间，将结果用数组存起来
// 时间复杂度O(N)，空间复杂度O(N)
func getKthFromEnd(head *ListNode, k int) *ListNode {
    p, n := head, 0
    trace := []*ListNode{}
    for p != nil {
        trace = append(trace, p)
        n ++
        p = p.Next
    }
    if n-k < 0 {
        return nil
    }
    return trace[n-k]   // n-k: 数组下标与题目标号方式相差1
}

// 遍历一次思路：快慢双指针：根据隐藏条件：倒数第k个节点与链表最后一个节点之间的距离是固定的k-1
// 时间复杂度O(N), 空间复杂度O(1)
func getKthFromEnd(head *ListNode, k int) *ListNode {
    p1, p2, n := head, head, 0
    // 鲁棒性
    if k == 0 || p1 == nil {
        return nil
    }
    for n < k-1 && p1 != nil {
        p1 = p1.Next
        n ++
    }
    // 鲁棒性：如果总节点个数 < k 那么走k-1步会导致空指针，所以需要判断 
    if p1 == nil {
        return nil
    }
    for p1 != nil {
        p1 = p1.Next
        if p1 == nil {      // 一旦p1到达末尾，立刻结束
            break
        }
        p2 = p2.Next
    }
    return p2
}
```

