---
title: 剑指OfferII027.回文链表
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-03-17 11:13:37
---

## 题目描述

[剑指 Offer II 027. 回文链表](https://leetcode-cn.com/problems/aMhZSa/)

难度简单

给定一个链表的 **头节点** `head` **，**请判断其是否为回文链表。

如果一个链表是回文，那么链表节点序列从前往后看和从后往前看是相同的。

<!-- more --> 

**示例 1：**

**![img](http://xwjpics.gumptlu.work/qinniu_uPic/1626421737-LjXceN-image.png)**

```
输入: head = [1,2,3,3,2,1]
输出: true
```

**示例 2：**

**![img](http://xwjpics.gumptlu.work/qinniu_uPic/1626422231-wgvnWh-image.png)**

```
输入: head = [1,2]
输出: false
```

**提示：**

- 链表 L 的长度范围为 `[1, 105]`
- `0 <= node.val <= 9`

**进阶：**能否用 O(n) 时间复杂度和 O(1) 空间复杂度解决此题？

## 解题思路及代码

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func isPalindrome(head *ListNode) bool {
    // 统计总长度
    length := 0
    for p:=head; p!=nil; p=p.Next {
        length++
    }
    if length == 1 {
        return true
    }
    // 找到前半部分翻转的结束节点位置
    filpOverNode := head
    for i:=0; i<length/2; i++ {
        filpOverNode = filpOverNode.Next
    }
    // 翻转前半部分
    flipHead := flipList(head, filpOverNode)
    // 判断回文串
    ans := true
    var p1, p2 *ListNode
    if length % 2 == 0 {        // 注意奇偶
        p1, p2 = filpOverNode, flipHead
    }else {
        p1, p2 = filpOverNode.Next, flipHead
    }
    for ; p1 != nil && p2 != nil; p1, p2 = p1.Next, p2.Next {
        if p1.Val != p2.Val {
            ans = false
            break
        }
    }
    // 恢复链表
    flipList(flipHead, nil)
    flipHead.Next = filpOverNode
    return ans
}

func flipList(head, end *ListNode) *ListNode {
    if head == nil || head.Next == nil {
        return head
    }
    // 快慢指针移动
    slow, fast := head, head.Next 
    // 翻转前半部分
    for fast != end {
        t := fast.Next
        fast.Next = slow
        slow = fast
        fast = t
    }
    head.Next = nil
    return slow
}
```

二刷：

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func isPalindrome(head *ListNode) bool {
    if head == nil || head.Next == nil {
        return true
    }

    sum := 0
    p := head
    for p != nil {
        sum ++
        p = p.Next
    }
    
    half := sum/2-1
    p = head
    for i:=0; i<half; i++ {
        p = p.Next
    }

    // 判断奇偶
    var t1, t2 *ListNode
    if sum % 2 == 0 {
        t1, t2 = p, p.Next
    }else {
        t1, t2 = p, p.Next.Next
    }

    // 断开
    t1.Next = nil
    // 翻转前半部分
    tmp1, tmp2 := t1, t2        // 记录
    t1 = filp(head)
    defer func() {
        // 复原
        filp(tmp1)
        tmp1.Next = tmp2
    }()
    for t1 != nil && t2 != nil {
        if t1.Val != t2.Val {
            return false
        }

        t1 = t1.Next
        t2 = t2.Next
    }

    if !(t1 == nil && t2 == nil) {
        return false
    }

    return true
}

func filp(head *ListNode) *ListNode {
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

    head.Next = nil
    return p1
}
```

