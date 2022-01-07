---
title: 剑指Offer23.链表中环的入口节点
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-14 10:11:27
---

## 题目描述

[剑指 Offer II 022. 链表中环的入口节点](https://leetcode-cn.com/problems/c32eOV/)

难度中等4

给定一个链表，返回链表开始入环的第一个节点。 从链表的头节点开始沿着 `next` 指针进入环的第一个节点为环的入口节点。如果链表无环，则返回 `null`。

为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。**注意，`pos` 仅仅是用于标识环的情况，并不会作为参数传递到函数中。**

**说明：**不允许修改给定的链表。



<!-- more --> 

**示例 1：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/circularlinkedlist.png)

```
输入：head = [3,2,0,-4], pos = 1
输出：返回索引为 1 的链表节点
解释：链表中有一个环，其尾部连接到第二个节点。
```

**示例 2：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/circularlinkedlist_test2.png)

```
输入：head = [1,2], pos = 0
输出：返回索引为 0 的链表节点
解释：链表中有一个环，其尾部连接到第一个节点。
```

**示例 3：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/circularlinkedlist_test3.png)

```
输入：head = [1], pos = -1
输出：返回 null
解释：链表中没有环。
```

 

**提示：**

- 链表中节点的数目范围在范围 `[0, 104]` 内
- `-105 <= Node.val <= 105`
- `pos` 的值为 `-1` 或者链表中的一个有效索引

**进阶：**是否可以使用 `O(1)` 空间解决此题？

## 解题思路及代码

### 1. 一般的思路

用空间记录节点浏览记录，判断重复判断入口

### 2. 快慢指针

* 确定是否有环：快慢指针是否重合

* 确定环大小：如果有环，那么在第一步的基础上在环中记录个数即可统计环的大小（节点数量）

* 确定环首部：快慢指针之间的差值固定为链表中环的大小，先让快指针走该大小的步数，然后快、慢指针一起走，重合时就是环的入口

==快慢指针之间的**固定间距**往往可以解决问题==

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */

// 遍历,记录次数
// 时间复杂度O(N)，空间复杂度O(N)
func detectCycle(head *ListNode) *ListNode {
    p, accountMap := head, make(map[*ListNode]int)
    for p != nil {
        if _, has := accountMap[p]; !has {
            accountMap[p] = 1
        }else {
            return p
        }
        p = p.Next
    }
    return nil
}

// 快慢指针
// 时间复杂度O(N)，空间复杂度O(1)
func detectCycle(head *ListNode) *ListNode {
    fast, low, n := head, head, 1
    for fast != nil && low != nil && fast.Next != nil{
        fast = fast.Next.Next
        low = low.Next
        if low != nil && fast != nil && low == fast {
            // 追上,有环,计算环中节点数量n
            for {
                fast = fast.Next
                if fast != low {
                    n ++
                }else {
                    break
                }
            } 
            break           
        }
    }
    // 无环
    if fast == nil || low == nil || fast.Next == nil {
        return nil
    }
    //双指针确定环首
    low = head
    for n > 0 {
        fast = fast.Next
        n --
    }
    for low != fast {
        low = low.Next
        fast = fast.Next
    }
    return low
}
```

