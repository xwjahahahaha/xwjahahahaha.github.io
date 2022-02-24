---
title: 剑指Offer33.二叉搜索树的后序遍历序列
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-02-23 23:05:50
---

## 题目描述

[剑指 Offer 33. 二叉搜索树的后序遍历序列](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/)

难度中等435

输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 `true`，否则返回 `false`。假设输入的数组的任意两个数字都互不相同。

 <!-- more -->

参考以下这颗二叉搜索树：

```
     5
    / \
   2   6
  / \
 1   3
```

**示例 1：**

```
输入: [1,6,3,2,5]
输出: false
```

**示例 2：**

```
输入: [1,3,2,6,5]
输出: true 
```

**提示：**

1. `数组长度 <= 1000`

## 解题思路及代码

```go
// 我的思路
// 排序数组得到二叉搜索树中序遍历结果
// 通过中序、后序遍历重构二叉树过程判断是否合法
// 时间复杂度O(N^2), 空间复杂度O(N)
func verifyPostorder(postorder []int) bool {
    // 排序得到中序遍历结果
    inorder := make([]int, len(postorder))
    for i, v := range postorder {
        inorder[i] = v
    }
    quickSort(inorder, 0, len(inorder)-1)
    
    // 递归使用构建二叉树的方法判断是否正确
    var canBuildTree func(inorder, postorder []int) bool
    canBuildTree = func(inorder, postorder []int) bool {
        // 递归结束条件
        m, n := len(inorder), len(postorder)
        if m == 0 && n == 0 {
            return true
        }

        // 取中间节点，判断左右区域
        rootVal := postorder[len(postorder)-1]
        Index := getIndex(inorder, rootVal)

        // 判断中序、后序的左右区域是否对应
        if !isEqualNums(inorder[:Index], postorder[:Index]) {
            return false
        }
        if !isEqualNums(inorder[Index+1:], postorder[Index:n-1]) {
            return false
        }

        return canBuildTree(inorder[:Index], postorder[:Index]) && canBuildTree(inorder[Index+1:], postorder[Index:n-1])
    }

    return canBuildTree(inorder, postorder)
}


func quickSort(nums []int, l, r int) {
    if l < r {
        mid := getMid(nums, l, r)
        quickSort(nums, l, mid-1)
        quickSort(nums, mid+1, r)
    }
}

func getMid(nums []int, l, r int) int {
    pivot := nums[l]
    for l < r {
        for l < r && nums[r] >= pivot { r-- }
        nums[l] = nums[r]
        for l < r && nums[l] <= pivot { l++ }
        nums[r] = nums[l]
    }
    
    nums[l] = pivot
    return l
}

func getIndex(nums []int, target int) int {
    for i, v := range nums {
        if v == target {
            return i
        }
    }

    return -1
}

// 判断两个数组元素是否相同
func isEqualNums(nums1, nums2 []int) bool {
    map1 := make(map[int]bool)
    for _, v := range nums1 {
        map1[v] = true
    }

    for _, v := range nums2 {
        if map1[v] == false {
            return false
        }
    }

    return true
}


// 根据后序遍历以及二叉搜索树的规则递归判断
// 时间复杂度O(N^2), 空间复杂度O(N)
func verifyPostorder(postorder []int) bool {
    n := len(postorder)
    // 递归结束条件
    if n == 0 {
        return true
    }

    // 找到当前root节点值
    rootVal := postorder[n-1]
    
    // 从左往右遍历找到第一个大于根节点的值就是右子树的开端
    rightIdx := n-1                 // 注意：这里要初始化为root的下标，因为可能没右子树
    for i:=0; i<n-1; i++ {
        if postorder[i] > rootVal {
            rightIdx = i
            break
        }
    }

    // [:rightIdx] 左子树一定大于根节点值，只需要判断右子树
    for i:=rightIdx; i<n; i++ {
        if postorder[i] < rootVal {
            return false
        }
    }

    // 递归左右子树
    return verifyPostorder(postorder[:rightIdx]) && verifyPostorder(postorder[rightIdx:n-1])
}
```

