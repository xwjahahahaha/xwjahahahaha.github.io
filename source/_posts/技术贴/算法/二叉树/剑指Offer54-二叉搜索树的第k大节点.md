---
title: 剑指Offer54.二叉搜索树的第k大节点
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-07-11 22:50:14
---

## 题目描述

[剑指 Offer 54. 二叉搜索树的第k大节点](https://leetcode.cn/problems/er-cha-sou-suo-shu-de-di-kda-jie-dian-lcof/)

难度简单

给定一棵二叉搜索树，请找出其中第 `k` 大的节点的值。

 <!-- more -->

**示例 1:**

```
输入: root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
输出: 4
```

**示例 2:**

```
输入: root = [5,3,6,2,4,null,null,1], k = 3
       5
      / \
     3   6
    / \
   2   4
  /
 1
输出: 4
```

**限制：**

- 1 ≤ k ≤ 二叉搜索树元素个数

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
// 二叉搜索树的中序遍历是有序数组
func kthLargest(root *TreeNode, k int) int {
    ans := -1
    var help func(*TreeNode)
    help = func(root *TreeNode) {
        if root == nil {
            return
        }
        help(root.Right)        // 先右再左，找第k大的
        k--
        if k == 0 {
            ans = root.Val      // 记录
            return              // 提前结束
        }
        help(root.Left)
    }
    help(root)

    return ans
}
```

二刷：

注意二叉搜索树特性：先左再右=>递增， 先右再左=>递减

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
// 二叉搜索树的中序遍历是排序数组
// 先左再右=>递增， 先右再左=>递减
func kthLargest(root *TreeNode, k int) int {
    res := -1
    var help func(root *TreeNode)
    help = func(root *TreeNode) {
        if root == nil {
            return 
        }

        help(root.Right)
        k-- 
        if k == 0 {
            res = root.Val
            return
        }
        help(root.Left)
    }

    help(root)
    return res
}
```

