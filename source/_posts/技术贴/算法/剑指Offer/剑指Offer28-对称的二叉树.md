---
title: 剑指Offer28.对称的二叉树
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-17 15:25:39
---

## 题目描述

[剑指 Offer 28. 对称的二叉树](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/)

难度简单

请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

例如，二叉树 [1,2,2,3,4,4,3] 是对称的。

`  1  / \ 2  2 / \ / \3  4 4  3`
但是下面这个 [1,2,2,null,3,null,3] 则不是镜像对称的:

```
  1  / \ 2  2  \  \  3   3
```

 

**示例 1：**

```
输入：root = [1,2,2,3,4,4,3]
输出：true
```

**示例 2：**

```
输入：root = [1,2,2,null,3,null,3]
输出：false
```

 

**限制：**

```
0 <= 节点个数 <= 1000
```

注意：本题与主站 101 题相同：https://leetcode-cn.com/problems/symmetric-tree/


<!-- more -->

## 解题思路及代码

核心思路：对称二叉树的前序遍历与修改的前序遍历(中->右->左)是相同的。

注意点：

1. 对于左右仅有一个子树的节点来说，空的子节点也需要设置一个特殊值放入遍历结果中（本题使用-1）
2. go语言中，函数传递切片时，如果函数中使用了append，则一定要传递切片的指针，否则一旦append越界拓展，原切片不会改动。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func isSymmetric(root *TreeNode) bool {
    ft, mft := make([]int, 0), make([]int, 0)
    // 如果对称则两种遍历结果相同
    first_traverse(root, &ft)   // 注意2: 0长度切片作为函数参数传递时，函数中使用append要传递指针，否则append后是新的切片，原切片不会改变
    modify_first_traverse(root, &mft) 
    for i, v := range ft {
        if mft[i] != v {
            return false
        }
    }   
    return true
}

// 中->左->右
func first_traverse(root *TreeNode, ft *[]int) {
    if root == nil {
        // 注意1：当一边为空子树的时候也要加入遍历数组中
        *ft = append(*ft, -1)
        return
    }
    *ft = append(*ft, root.Val)
    if root.Left != nil || root.Right != nil {
        first_traverse(root.Left, ft)
        first_traverse(root.Right, ft)
    }
}

// 中->右->左
func modify_first_traverse(root *TreeNode, mft *[]int) {
    if root == nil {
        *mft = append(*mft, -1)
        return
    }
    *mft = append(*mft, root.Val)
    if root.Left != nil || root.Right != nil {
        modify_first_traverse(root.Right, mft)
        modify_first_traverse(root.Left, mft)   
    }
}
```

