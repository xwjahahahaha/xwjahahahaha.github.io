---
title: 剑指Offer32-I.从上到下打印二叉树
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-23 13:13:46
---

## 题目描述

[剑指 Offer 32 - I. 从上到下打印二叉树](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)

难度中等

从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

 <!-- more -->

例如:
给定二叉树: `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回：

```
[3,9,20,15,7]
```

 

**提示：**

1. `节点总数 <= 1000`

## 解题思路及代码

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
// 二叉树层次遍历
func levelOrder(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    queue, ret, levelFlag := []*TreeNode{}, []int{}, &TreeNode{-1, nil, nil}
    // 加入头节点、层结束标记
    queue = append(queue, root)
    queue = append(queue, levelFlag)
    for len(queue) > 0 {
        // 出节点
        node := queue[0]
        fmt.Println(node.Val)
        queue = queue[1:]
        if node != levelFlag {
            ret = append(ret, node.Val)
        }
        // 节点左右孩子入队
        if node.Left != nil {
            queue = append(queue, node.Left)
        }
        if node.Right != nil {
            queue = append(queue, node.Right)
        }
        // 添加层结束标记
        if len(queue) > 1 && queue[0] == levelFlag {        // 如果最后队列中只有一个层结束标记，那么就
            // 如果当前队列中下一个是层结束标记节点，那么表示此节点是当前层最后一个节点，其右孩子就是下一层的最后一个节点, 添加层结束标记
            queue = append(queue, levelFlag)
        }
    }
    return ret
}
```

