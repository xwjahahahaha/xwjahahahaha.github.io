---
title: 剑指Offer34.二叉树中和为某一值的路径
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2021-12-17 14:17:48
---

## 题目描述

[剑指 Offer 34. 二叉树中和为某一值的路径](https://leetcode-cn.com/problems/er-cha-shu-zhong-he-wei-mou-yi-zhi-de-lu-jing-lcof/)

难度中等

给你二叉树的根节点 `root` 和一个整数目标和 `targetSum` ，找出所有 **从根节点到叶子节点** 路径总和等于给定目标和的路径。

**叶子节点** 是指没有子节点的节点。

 <!-- more -->

**示例 1：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/pathsumii1.jpg)

```
输入：root = [5,4,8,11,null,13,4,7,2,null,null,5,1], targetSum = 22
输出：[[5,4,11,2],[5,8,4,5]]
```

**示例 2：**

![img](http://xwjpics.gumptlu.work/qinniu_uPic/pathsum2.jpg)

```
输入：root = [1,2,3], targetSum = 5
输出：[]
```

**示例 3：**

```
输入：root = [1,2], targetSum = 0
输出：[]
```

 

**提示：**

- 树中节点总数在范围 `[0, 5000]` 内
- `-1000 <= Node.val <= 1000`
- `-1000 <= targetSum <= 1000`

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
// 回溯，深度优先遍历树（先序遍历）
// 注意不用剪枝：因为节点的值可能为负数
func pathSum(root *TreeNode, target int) [][]int {
    ret := make([][]int, 0)
    var dfs func(*TreeNode, int)
    path := make([]int, 0)
    // 深度优先遍历树 left表示剩余的值
    dfs = func(node *TreeNode, left int) {
        if node == nil {
            return 
        }
        // 计算本节点
        left -= node.Val
        path = append(path, node.Val)
        // 判断是否是叶子结点并满足和
        if node.Left == nil && node.Right == nil && left == 0 {
            // 加入结果集合
            ret = append(ret, append([]int{}, path...))        // 复制一个新数组切片的方法（小技巧）: append([]int{}, path...)
        }
        // 递归左右孩子
        dfs(node.Left, left)
        dfs(node.Right, left)
        // 回溯，将当前节点退出path
        path = path[:len(path)-1]
    }
    dfs(root, target)
    return ret
}
```

