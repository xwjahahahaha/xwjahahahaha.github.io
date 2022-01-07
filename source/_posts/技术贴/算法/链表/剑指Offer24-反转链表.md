---
title: 剑指Offer24.反转链表
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-14 14:45:53
---

## 题目描述

[剑指 Offer 24. 反转链表](https://leetcode-cn.com/problems/fan-zhuan-lian-biao-lcof/)

难度简单

定义一个函数，输入一个链表的头节点，反转该链表并输出反转后链表的头节点。

 

**示例:**

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

 

**限制：**

```
0 <= 节点个数 <= 5000
```

 

**注意**：本题与主站 206 题相同：https://leetcode-cn.com/problems/reverse-linked-list/




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
// 栈，直接的思路
// 时间复杂度O(N),空间复杂度O(N)
func reverseList(head *ListNode) *ListNode {
    if head == nil {
        return nil
    }
    record, p := []*ListNode{}, head
    for p != nil {
        record = append(record, p)
        p = p.Next
    }
    p1, p2 := record[len(record)-1], record[len(record)-1]
    for len(record) > 0 {
        node := record[len(record)-1]
        p1.Next = node
        p1 = p1.Next
        record = record[:len(record)-1]
    }
    p1.Next = nil
    return p2
}

// 借助三个指针原地翻转指针指向
// 时间复杂度O(N),空间复杂度O(1)
func reverseList(head *ListNode) *ListNode {
    p1, p2, p3 := head, head, head      // p2为当前考量节点,p1为其前置节点,p3为其后置节点
    if p1 == nil {
        // 空链表
        return nil
    }else if p1.Next == nil {
        // 仅有一个节点
        return head
    }else if p1.Next.Next == nil {
        // 仅有两个节点
        p2 = p2.Next
        p1.Next = nil
        p2.Next = p1
        return p2
    }
    // 三个以上节点
    for p2 != nil {
        if p2 == head {
            // 第一个节点
            p2 = p2.Next
            p3 = p2.Next
            p1.Next = nil
        }else if p2.Next == nil {
            // 最后一个节点，返回结果
            p2.Next = p1
            return p2
        } else {
            // 所有中间节点调换顺序
            // 调换当前顺序
            p2.Next = p1
            // 后移一个节点位置(移动三个指针)
            p1 = p2
            p2 = p3
            p3 = p2.Next
        }
    }
    return nil
}
```

直观/简单的思路：


```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    p1, p2 := head, head.Next
    for p2 != nil {
        t := p2.Next
        p2.Next = p1
        p1 = p2
        p2 = t
    }
    // 注意将第一个节点指针取消
    head.Next = nil
    return p1
}
```

