---
title: 剑指Offer18.删除链表的节点
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-07-22 10:42:09
---

## 题目描述

[剑指 Offer 18. 删除链表的节点](https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof/)

难度简单

给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。

返回删除后的链表的头节点。

<!-- more -->

**注意：**此题对比原题有改动

**示例 1:**

```
输入: head = [4,5,1,9], val = 5
输出: [4,1,9]
解释: 给定你链表中值为 5 的第二个节点，那么在调用了你的函数之后，该链表应变为 4 -> 1 -> 9.
```

**示例 2:**

```
输入: head = [4,5,1,9], val = 1
输出: [4,5,9]
解释: 给定你链表中值为 1 的第三个节点，那么在调用了你的函数之后，该链表应变为 4 -> 5 -> 9.
```

 

**说明：**

- 题目保证链表中节点的值互不相同
- 若使用 C 或 C++ 语言，你不需要 `free` 或 `delete` 被删除的节点

## 解题思路及代码

本题是入门题，相信一般的解法不会难到大家

```go
// 直接的思路
// 时间复杂度O(N)
func deleteNode(head *ListNode, val int) *ListNode {
    if head.Val == val {
        return head.Next    // 即使只有一个头节点，刚好返回nil
    }
    p := head
    for p != nil {
        if p.Next != nil && p.Next.Val == val {
            if p.Next.Next == nil {
                p.Next = nil
            }else {
                p.Next = p.Next.Next
            }
            return head
        }
        p = p.Next
    }
    return head
}
```

但是需要学习的是还有一种给参方式：参数给的不是要删除的Val而是对应链表中节点的指针（也就是不用我们去遍历找了）

那在这种情况下怎样删除该节点呢？

主要的问题在于：**单向链表中无法从当前节点获取上一个节点**

具体的思路可以这样解决：(来自剑指offer书)

![G1dn8u](http://xwjpics.gumptlu.work/qinniu_uPic/G1dn8u.png)

例如给出了i节点的指针，那么可以将j节点的值复制到i，然后将i原本的指针指向j的后继节点，最后删除j节点i即可

此方法相当于从头开始遍历找i的前节点时间复杂度仅为O(1)

需要注意的是，如果给出的i是最后一个末尾节点，那么还是需要从头开始遍历（因为没有后续节点了），所以时间复杂度为O(n)

所以此方法的平均时间复杂度为: [(n-1)\*1 + 1\*n] / n= O(1) 

