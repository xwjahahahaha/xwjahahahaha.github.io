---
title: 剑指Offer11.旋转数组的最小数字
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-19 14:45:41
---

## 题目描述

[剑指 Offer 11. 旋转数组的最小数字](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)

难度简单

把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个递增排序的数组的一个旋转，输出旋转数组的最小元素。例如，数组 `[3,4,5,1,2]` 为 `[1,2,3,4,5]` 的一个旋转，该数组的最小值为1。 

<!-- more -->

**示例 1：**

```
输入：[3,4,5,1,2]
输出：1
```

**示例 2：**

```
输入：[2,2,2,0,1]
输出：0
```

注意：本题与主站 154 题相同：https://leetcode-cn.com/problems/find-minimum-in-rotated-sorted-array-ii/

## 解题思路及代码

```go
// 遍历一遍即可找到答案
// 时间复杂度O(N), 不够好
func minArray(numbers []int) int {
    for i:=1; i<len(numbers); i++ {
        if numbers[i] < numbers[i-1] {
            return numbers[i]
        }
    }
    return numbers[0]
}

// 二分查找
// 按照题意: 1. 可以将数组分为两段数组, 前面一段升序, 后面一段升序, 前面一段元素都大于后面一段,中间的边界就是最小值的位置
// 每次取中间值, 如果mid与left或者right构成升序,那么就可以丢弃一段, 整个过程就是left不断成为前段升序的最后一个索引
// right就是后段的最后一个索引, 直到left + 1 == right 就找到了分界, 结束二分
func minArray(numbers []int) int {
    n := len(numbers)
    l, r := 0, n-1
    for numbers[l] >= numbers[r] {  // 等号也取, 相等时最小数字也有可能出现在中间, 例[1,0,1,1,1]
        mid := (l + r) >> 1
        if l + 1 == r {
            return numbers[r]
        }
        // 重点: 可能会出现num[left] == num[right] == num[mid]的情况, 此时无法判断中点属于哪个子有序序列
        // 但是结果一定在里面, 所以需要用顺序遍历
        if numbers[l] == numbers[r] && numbers[r] == numbers[mid] {
            for i:=l; i<r; i++ {
                if numbers[i] > numbers[i+1] {
                    return numbers[i+1]
                }
            }
            // 如果没找到, 那么就默认返回第一个, 因为有可能总长度就是1, 例[1]
            return numbers[l]
        } 
        if numbers[l] <= numbers[mid] {
            l = mid
        }else if numbers[mid] <= numbers[r]{
            r = mid
        }
    }
    return numbers[0]
}
```

