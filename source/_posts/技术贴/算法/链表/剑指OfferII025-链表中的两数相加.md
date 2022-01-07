---
title: 剑指OfferII025.链表中的两数相加
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2021-12-17 15:07:04
---

## 题目描述

[剑指 Offer II 025. 链表中的两数相加](https://leetcode-cn.com/problems/lMSNwu/)

难度中等

给定两个 **非空链表** `l1`和 `l2` 来代表两个非负整数。数字最高位位于链表开始位置。它们的每个节点只存储一位数字。将这两数相加会返回一个新的链表。

可以假设除了数字 0 之外，这两个数字都不会以零开头。

 <!-- more -->

**示例1：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/1626420025-fZfzMX-image.png)

```
输入：l1 = [7,2,4,3], l2 = [5,6,4]
输出：[7,8,0,7]
```

**示例2：**

```
输入：l1 = [2,4,3], l2 = [5,6,4]
输出：[8,0,7]
```

**示例3：**

```
输入：l1 = [0], l2 = [0]
输出：[0]
```

 

**提示：**

- 链表的长度范围为` [1, 100]`
- `0 <= node.val <= 9`
- 输入数据保证链表代表的数字无前导 0

 

**进阶：**如果输入链表不能修改该如何处理？换句话说，不能对列表中的节点进行翻转。

 

注意：本题与主站 445 题相同：https://leetcode-cn.com/problems/add-two-numbers-ii/



## 解题思路及代码

方法一：翻转解决逆序

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 链表翻转计算得出结果链表然后再翻转
// 缺点：会修改原来的链表
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    // 1. 翻转l1, l2
    head1 := flipList(l1)
    head2 := flipList(l2)
    // 2. 累加结果放到一个新的结果链表
    retHead, carry := &ListNode{-1, nil}, 0      // retHead是虚拟头节点, carry保存进位
    p1, p2, p3 := head1, head2, retHead
    for p1 != nil && p2 != nil {
        sum := p1.Val + p2.Val + carry
        carry = sum / 10
        // 链接新节点
        p3.Next = &ListNode{sum % 10, nil}
        p1, p2, p3 = p1.Next, p2.Next, p3.Next
    }
    // 处理剩余
    // 处理l1剩余
    for p1 != nil {
        sum := p1.Val + carry
        carry = sum / 10
        p3.Next = &ListNode{sum % 10, nil}
        p1, p3 = p1.Next, p3.Next
    }
    // 处理l2剩余
    for p2 != nil {
        sum := p2.Val + carry
        carry = sum / 10
        p3.Next = &ListNode{sum % 10, nil}
        p2, p3 = p2.Next, p3.Next
    }
    // 处理carry剩余
    if carry != 0 {
        p3.Next = &ListNode{carry, nil}
    }
    // 3. 翻转结果链表，返回
    return flipList(retHead.Next)
}

// // 翻转链表
func flipList(head *ListNode) *ListNode {
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
    // 消除头节点的第一个指针
    head.Next = nil
    return p1
}
```
方法二：双栈解决逆序
```go
// 用两个栈存储链表值，让右端对齐
// 结果链表不需要翻转：每次用一个指针保存结果链表头部然后让新节点指向它再更新这个指针即可
func addTwoNumbers(l1 *ListNode, l2 *ListNode) *ListNode {
    stack1, stack2 := make([]int, 0), make([]int, 0)
    // 1. 读取到栈
    for p1 := l1; p1 != nil; p1 = p1.Next {
        stack1 = append(stack1, p1.Val)
    }
    for p2 := l2; p2 != nil; p2 = p2.Next {
        stack2 = append(stack2, p2.Val)
    }
    // 2. 出栈计算
    var retP *ListNode  // 始终指向结果链表的头节点
    carry := 0       
    for len(stack1) > 0 && len(stack2) > 0 {
        n1, n2 := stack1[len(stack1)-1], stack2[len(stack2)-1]
        stack1, stack2 = stack1[:len(stack1)-1], stack2[:len(stack2)-1]
        sum := n1 + n2 + carry
        carry = sum / 10
        node := &ListNode{sum % 10, retP}   // 指向旧链表头节点            
        retP = node
    }
    // 处理剩余
    for len(stack1) > 0 {
        n1 := stack1[len(stack1)-1]
        stack1 = stack1[:len(stack1)-1]
        sum := n1 + carry
        carry = sum / 10
        node := &ListNode{sum % 10, retP}
        retP = node
    }
    for len(stack2) > 0 {
        n2 := stack2[len(stack2)-1]
        stack2 = stack2[:len(stack2)-1]
        sum := n2 + carry
        carry = sum / 10
        node := &ListNode{sum % 10, retP}
        retP = node
    }
    if carry != 0 {
        node := &ListNode{carry % 10, retP}
        retP = node
    }
    return retP
}
```

