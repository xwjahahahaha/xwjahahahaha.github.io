---
title: 剑指Offer32-II.从上到下打印二叉树II
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-24 21:25:00
---

## 题目描述

[剑指 Offer 32 - II. 从上到下打印二叉树 II](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-ii-lcof/)

难度简单

从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

 

例如:
给定二叉树: `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回其层次遍历结果：

```
[
  [3],
  [9,20],
  [15,7]
]
```

 

**提示：**

1. `节点总数 <= 1000`

注意：本题与主站 102 题相同：https://leetcode-cn.com/problems/binary-tree-level-order-traversal/


<!-- more -->

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
// 层次遍历
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    queue, division, ret := make([]*TreeNode, 0), &TreeNode{-1, nil, nil}, make([][]int, 0)
    queue = append(queue, root)
    queue = append(queue, division)
    ret = append(ret, []int{})
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        if node == division {
            if len(queue) > 0 {     // 注意两个if条件不能直接合并
                // 该层结束，创建下一个层空间
                level := make([]int, 0)
                ret = append(ret, level)
                // 给下一层添加结束标记
                queue = append(queue, division)
            }
        }else {
            // 加入结果集合
            ret[len(ret)-1] = append(ret[len(ret)-1], node.Val)
            // 左右孩子入队
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
    }
    return ret
}
```

