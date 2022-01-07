---
title: 剑指Offer25.合并两个排序的链表
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-15 10:08:57
---

## 题目描述

[剑指 Offer 25. 合并两个排序的链表](https://leetcode-cn.com/problems/he-bing-liang-ge-pai-xu-de-lian-biao-lcof/)

难度简单

输入两个递增排序的链表，合并这两个链表并使新链表中的节点仍然是递增排序的。

**示例1：**

```
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

**限制：**

```
0 <= 链表长度 <= 1000
```

注意：本题与主站 21 题相同：https://leetcode-cn.com/problems/merge-two-sorted-lists/

<!-- more -->

## 解题思路及代码

借助额外空间的不在此讨论，这里使用本地合并

因为两个链表都是有序的，所以使用两个指针分别指向两个链表，谁头节点小则链接到合并链表，随后后移指针。整个过程是一个递归的过程:

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 原地修改
// 时间复杂度O(N),空间复杂度O(1)
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    newHead := &ListNode{-1, nil}
    mergeHeadNode(newHead, l1, l2)
    return newHead.Next
}

func mergeHeadNode(befor, p1, p2 *ListNode) {
    // 鲁棒性处理
    if p1 == nil {
        befor.Next = p2
        return 
    }else if p2 == nil {
        befor.Next = p1
        return 
    }
    // 比较两个头节点
    if p1.Val <= p2.Val {
        befor.Next = p1
        // 递归
        mergeHeadNode(befor.Next, p1.Next, p2)
    }else {
        befor.Next = p2
        // 递归
        mergeHeadNode(befor.Next, p1, p2.Next)
    }
}
```

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 迭代
// 链表合并考虑哑节点, 合并不需要创建多余空间而是指针的处理
// 时间复杂度O(M+N), 空间复杂度O(1)
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
   pre := &ListNode{}
   t := pre
   for l1 != nil && l2 != nil {
       if l1.Val <= l2.Val {
           // 将pre连接到较小的节点
           pre.Next = l1
           // l1前进一步
           l1 = l1.Next
       }else {
           pre.Next = l2
           // l2前进一步
           l2 = l2.Next
       }
       // pre前进一步
       pre = pre.Next
   }
   // 考虑有一方先结束
   if l1 == nil && l2 != nil {
       pre.Next = l2
   }
   if l2 == nil && l1 != nil {
       pre.Next = l1
   }
   return t.Next
}


// 递归
// 假设头节点l1小，则递归公式是： merge(l1, l2) = l1.Next = merge(l1.Next, l2) = ...  （自上而下的递归）
// 子问题与主问题有相同的结构
// 时间复杂度O(M+N), 空间复杂度O(M+N)
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    // 递归结束条件
    // 有一方是空则返回另一方指针
    if l1 == nil {
        return l2
    }else if l2 == nil {
        return l1
    }
    // 判断当前谁小
    if l1.Val < l2.Val {
        // 递归连接到下一个
        l1.Next = mergeTwoLists(l1.Next, l2)
        return l1
    }else {
        l2.Next =  mergeTwoLists(l1, l2.Next)
        return l2
    }
}
```

