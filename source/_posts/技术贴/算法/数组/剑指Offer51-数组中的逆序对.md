---
title: 剑指Offer51.数组中的逆序对
tags:
  - null
categories:
  - technical
  - leetcode
toc: true
date: 2022-03-02 11:15:35
---

## 题目描述

在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数。

 <!-- more -->

示例 1:

输入: [7,5,6,4]
输出: 5


限制：

0 <= 数组长度 <= 50000

## 解题思路及代码

* 利用`nums[p1]<=nums[p2]`时统计（两处）

```go
func reversePairs(nums []int) int {
	// 归并排序
	var sortNum func(l, r int) int
	sortNum = func(l, r int) int {
        if l >= r {
            return 0
        }
				mid := (r-l)/2 + l
        count := sortNum(l, mid) + sortNum(mid+1, r)
        if nums[mid] > nums[mid+1] {
            count += merge(nums, l, mid, r)
        }
        return count
	}

	return sortNum(0, len(nums)-1)
}

func merge(nums []int, l, mid, r int) int {
	help, count, t := make([]int, r-l+1), 0, 0
	p1, p2 := l, mid+1
	for p1 <= mid && p2 <= r {
		// 统计逆序对个数
		if nums[p1] <= nums[p2] {       // 注意这里是等于
			count += p2 - mid - 1
			help[t] = nums[p1]
			p1++
			t++
		} else {
			help[t] = nums[p2]
			p2++
			t++
		}
	}
	// 剩余
	for p1 <= mid {
		help[t] = nums[p1]
    count += r-(mid+1)+1        // 关键这里也要统计
		p1++
		t++
	}
	for p2 <= r {
		help[t] = nums[p2]
		p2++
		t++
	}
	// 拷贝
	for i := 0; i < len(help); i++ {
		nums[l+i] = help[i]
	}

	return count
}
```

* 使用`nums[p2]<nums[p1]`时统计（一处）

```go
func reversePairs(nums []int) int {
	// 归并排序
	var sortNum func(l, r int) int
	sortNum = func(l, r int) int {
        if l >= r {
            return 0
        }
		mid := (r-l)/2 + l
        count := sortNum(l, mid) + sortNum(mid+1, r)
        if nums[mid] > nums[mid+1] {
            count += merge(nums, l, mid, r)
        }
        return count
	}

	return sortNum(0, len(nums)-1)
}

func merge(nums []int, l, mid, r int) int {
	help, count, t := make([]int, r-l+1), 0, 0
	p1, p2 := l, mid+1
	for p1 <= mid && p2 <= r {
		// 统计逆序对个数
		if nums[p1] <= nums[p2] {       // 注意这里是等于
			// count += p2 - mid - 1
			help[t] = nums[p1]
			p1++
			t++
		} else {
			help[t] = nums[p2]
      // 如果使用p2就是nums[p1]>nums[p2], p1~mid的所有数都与nums[p2]构成逆序对
      count += mid-p1+1       
			p2++
			t++
		}
	}
	// 剩余
	for p1 <= mid {
		help[t] = nums[p1]
    // count += r-(mid+1)+1        // 关键
		p1++
		t++
	}
	for p2 <= r {
    // 如果使用p2此处不用球，因为当p1==mid就没有后续<nums[p2]的数了（此时的nums[p1]已经在上面统计过了）
		help[t] = nums[p2]          
		p2++
		t++
	}
	// 拷贝
	for i := 0; i < len(help); i++ {
		nums[l+i] = help[i]
	}

	return count
}
```

