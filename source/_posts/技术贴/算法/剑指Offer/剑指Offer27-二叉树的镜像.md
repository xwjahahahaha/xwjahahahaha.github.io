---
title: 剑指Offer27.二叉树的镜像
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-17 11:08:57
---

## 题目描述

[剑指 Offer 27. 二叉树的镜像](https://leetcode-cn.com/problems/er-cha-shu-de-jing-xiang-lcof/)

请完成一个函数，输入一个二叉树，该函数输出它的镜像。

例如输入：

     4
   /   \
  2     7
 / \   / \
1   3 6   9
镜像输出：

     4
   /   \
  7     2
 / \   / \
9   6 3   1

 

示例 1：

输入：root = [4,2,7,1,3,6,9]
输出：[4,7,2,9,6,3,1]


限制：

0 <= 节点个数 <= 1000

注意：本题与主站 226 题相同：https://leetcode-cn.com/problems/invert-binary-tree/


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
func mirrorTree(root *TreeNode) *TreeNode {
    exchange(root)
    return root
}

// 先序遍历同时交换左右子树
func exchange(root *TreeNode) {
    if root == nil {
        return 
    }
    root.Left, root.Right = root.Right, root.Left
    exchange(root.Left)
    exchange(root.Right)
}
```

