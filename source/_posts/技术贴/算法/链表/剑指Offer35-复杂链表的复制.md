---
title: 剑指Offer35.复杂链表的复制
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-04-28 12:27:13
---

## 题目描述

[剑指 Offer 35. 复杂链表的复制](https://leetcode-cn.com/problems/fu-za-lian-biao-de-fu-zhi-lcof/)

难度中等

请实现 `copyRandomList` 函数，复制一个复杂链表。在复杂链表中，每个节点除了有一个 `next` 指针指向下一个节点，还有一个 `random` 指针指向链表中的任意节点或者 `null`。

<!-- more -->

**示例 1：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/e1.png)

```
输入：head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
输出：[[7,null],[13,0],[11,4],[10,2],[1,0]]
```

**示例 2：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/e2.png)

```
输入：head = [[1,1],[2,1]]
输出：[[1,1],[2,1]]
```

**示例 3：**

**![img](http://xwjpics.gumptlu.work/qinniu_uPic/e3.png)**

```
输入：head = [[3,null],[3,0],[3,null]]
输出：[[3,null],[3,0],[3,null]]
```

**示例 4：**

```
输入：head = []
输出：[]
解释：给定的链表为空（空指针），因此返回 null。
```

**提示：**

- `-10000 <= Node.val <= 10000`
- `Node.random` 为空（null）或指向链表中的节点。
- 节点数目不超过 1000 。 

**注意：**本题与主站 138 题相同：https://leetcode-cn.com/problems/copy-list-with-random-pointer/ 

## 解题思路及代码

```go
/**
 * Definition for a Node.
 * type Node struct {
 *     Val int
 *     Next *Node
 *     Random *Node
 * }
 */

var cacheNode map[*Node]*Node   // 旧node => 新node 记录每一节点对应新节点的创建情况

// 方法一：回溯/递归拷贝 + hash防止重复拷贝
// 时间复杂度O(N) 空间复杂度O(N) 每个节点最多被Next、Random指针访问两次
func copyRandomList(head *Node) *Node {
    cacheNode = make(map[*Node]*Node)
    return deepCopy(head)
}

func deepCopy(node *Node) *Node {
    if node == nil {
        return nil
    }    
    // 判断是否已经创建
    if newNode, has := cacheNode[node]; has {
        return newNode
    } 
    // 拷贝
    newNode := &Node{Val: node.Val}
    // 注意：在此处就应该记录，因为已经创建（虽然指针还未填写）
    cacheNode[node] = newNode
    newNode.Next = deepCopy(node.Next)
    newNode.Random = deepCopy(node.Random)
    return newNode
}
```
方法二看官方题解ppt

```go
// 方法二：在原链表上复制一个重复的节点并连接，用链表的连接关系替换掉hash表的额外空间 
func copyRandomList(head *Node) *Node {
    if head == nil {
        return nil
    }
    // 对于每个节点S，复制一个后继节点S'
    for node := head; node != nil; node = node.Next.Next {
        copyNode := &Node{Val: node.Val, Next: node.Next}
        node.Next = copyNode
    }
    // 处理所有复制节点的random指针
    for node := head; node != nil; node = node.Next.Next {
        copyNode := node.Next
        // 注意也必须指向的是random指向节点的复制节点
        // 注意node.Random可能是nil
        if node.Random != nil {
            copyNode.Random = node.Random.Next
        }   
    }
    // 分离两个链表
    newHead := head.Next
    for node := head; node != nil; node = node.Next {   // 注意这里Next已经改变
        copyNode := node.Next
        // 注意：应该先调整node的next再调整copyNode的next，反了会导致node.Next.Next改变（因为copyNode的next变了）
        node.Next = node.Next.Next
        if copyNode.Next != nil {
            copyNode.Next = copyNode.Next.Next
        }
        
    }
    return newHead
}
```

