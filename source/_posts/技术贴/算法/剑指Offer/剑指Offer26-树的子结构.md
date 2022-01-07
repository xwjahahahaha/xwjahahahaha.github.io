---
title: 剑指Offer26.树的子结构
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-16 11:24:18
---

## 题目描述

[剑指 Offer 26. 树的子结构](https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/)

难度中等353

输入两棵二叉树A和B，判断B是不是A的子结构。(约定空树不是任意一个树的子结构)

B是A的子结构， 即 A中有出现和B相同的结构和节点值。

例如:
给定的树 A:

![RiBSdM](http://xwjpics.gumptlu.work/qinniu_uPic/RiBSdM.png)
给定的树 B：

![OzDGlO](http://xwjpics.gumptlu.work/qinniu_uPic/OzDGlO.png)`

返回 true，因为 B 与 A 的一个子树拥有相同的结构和节点值。

**示例 1：**

```
输入：A = [1,2,3], B = [3,1]
输出：false
```

**示例 2：**

```
输入：A = [3,4,5,1,2], B = [4,1]
输出：true
```

**限制：**

```
0 <= 节点个数 <= 10000
```

<!-- more -->

## 解题思路及代码

剖解为两个问题：

1. 遍历A中找到与B匹配的根节点（可能不只一个） => 递归遍历，一旦符合条件进入第二部分
2. 对于匹配的两个根节点，判断子树是否匹配.   =>  递归左右子节点

注意点：

1. 鲁棒性：判断当前节点是否为空而不是去判断左右孩子节点是否可能为空（更好理解）
2. 第一个问题要考察所有匹配点都为false才false，第二个问题对于单个匹配点一但有false就直接返回false，不同的思路，不同的判断。

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func isSubStructure(A *TreeNode, B *TreeNode) bool {
   res := false
   if A != nil && B != nil {
        if A.Val == B.Val {
            // 找到根节点
            res = DoesTreeAHaveTreeB(A, B)
        }
        // 如果为false则继续寻找左子树
        if !res {
            res = isSubStructure(A.Left, B)
        }
        // 如果为false则继续寻找右子树
        if !res {
            res = isSubStructure(A.Right, B)
        }
   }
   return res
}

// 判断子树
func DoesTreeAHaveTreeB(A *TreeNode, B *TreeNode) bool {
    if B == nil {
        // 如果B节点是空则一定匹配
        return true
    }
    if A == nil {
        // 如果B不为空且A为空则一定不匹配
        return false
    }
    if A.Val != B.Val {
        // 如果两节点都不为空且值不相同也一定为false
        return false
    }
    // 如果相同则继续迭代子树
    return DoesTreeAHaveTreeB(A.Left, B.Left) && DoesTreeAHaveTreeB(A.Right, B.Right)
}
```

