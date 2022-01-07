---
title: 剑指Offer30.包含min函数的栈
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-09-20 15:50:03
---

## 题目描述

[剑指 Offer 30. 包含min函数的栈](https://leetcode-cn.com/problems/bao-han-minhan-shu-de-zhan-lcof/)

难度简单

定义栈的数据结构，请在该类型中实现一个能够得到栈的最小元素的 min 函数在该栈中，调用 min、push 及 pop 的时间复杂度都是 O(1)。

 

**示例:**

```
MinStack minStack = new MinStack();
minStack.push(-2);
minStack.push(0);
minStack.push(-3);
minStack.min();   --> 返回 -3.
minStack.pop();
minStack.top();      --> 返回 0.
minStack.min();   --> 返回 -2.
```

 

**提示：**

1. 各函数的调用总次数不超过 20000 次

 

注意：本题与主站 155 题相同：https://leetcode-cn.com/problems/min-stack/


<!-- more -->

## 解题思路及代码

使用辅助栈记录当前数据栈的状态

时间复杂度O(1)，空间复杂度O(N)

```go
type MinStack struct {
    stack []int
    minStack []int      // 辅助记录当前最小值的栈       
}

const INT_MAX = int(^uint(0) >> 1)

/** initialize your data structure here. */
func Constructor() MinStack {
    return MinStack{
        stack : make([]int, 0),
        minStack : make([]int, 0),
    }
}

func (this *MinStack) Push(x int)  {
    // 更新辅助栈
    // 比较辅助栈中栈顶记录的上一次的最小值与新插入的值x值之间的大小，决定本次最小值
    n := len(this.minStack)
    if n == 0 || x < this.minStack[n-1] {
        this.minStack = append(this.minStack, x)
    }else {
        this.minStack = append(this.minStack, this.minStack[n-1])
    }
    // 更新数据栈
    this.stack = append(this.stack, x)
}

// 删除栈顶元素，不返回
func (this *MinStack) Pop()  {
    this.stack = this.stack[:len(this.stack)-1]
    // 辅助栈一直保存每一个新数插入时的状态（最小值），所以当删除一个数时同步删除一个状态即可回到上一个状态
    this.minStack = this.minStack[:len(this.minStack)-1]
}

// 不删除栈顶元素，返回值
func (this *MinStack) Top() int {
    return this.stack[len(this.stack)-1]
}


func (this *MinStack) Min() int {
    return this.minStack[len(this.minStack)-1]
}


/**
 * Your MinStack object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Push(x);
 * obj.Pop();
 * param_3 := obj.Top();
 * param_4 := obj.Min();
 */
```

方法二：不允许使用额外的空间

```go
```

