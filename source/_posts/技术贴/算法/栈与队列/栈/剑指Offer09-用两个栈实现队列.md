---
title: 剑指Offer09.用两个栈实现队列
tags:
  - golang
categories:
  - technical
  - leetcode
toc: true
date: 2021-06-10 10:47:46
---

## 题目描述

[剑指 Offer 09. 用两个栈实现队列](https://leetcode-cn.com/problems/yong-liang-ge-zhan-shi-xian-dui-lie-lcof/)

难度简单

用两个栈实现一个队列。队列的声明如下，请实现它的两个函数 `appendTail` 和 `deleteHead` ，分别完成在队列尾部插入整数和在队列头部删除整数的功能。(若队列中没有元素，`deleteHead` 操作返回 -1 )

 <!-- more -->

**示例 1：**

```
输入：
["CQueue","appendTail","deleteHead","deleteHead"]
[[],[3],[],[]]
输出：[null,null,3,-1]
```

**示例 2：**

```
输入：
["CQueue","deleteHead","appendTail","appendTail","deleteHead","deleteHead"]
[[],[],[5],[2],[],[]]
输出：[null,-1,null,null,5,2]
```

**提示：**

- `1 <= values <= 10000`
- `最多会对 appendTail、deleteHead 进行 10000 次调用`

## 解题思路及代码

```go
type CQueue struct {
    stack1 []int
    stack2 []int
}


func Constructor() CQueue {
    return CQueue{[]int{},[]int{}}
}


func (this *CQueue) AppendTail(value int)  {
    this.stack1 = append(this.stack1, value)
}


func (this *CQueue) DeleteHead() int {
    n1, n2 := len(this.stack1), len(this.stack2)
    if n2 == 0 {        // 注意; 只有stack2为空才转移
        // 栈2长度为0, 则转移栈1到栈2
        for n1 > 0 {
            // 不为空并且当栈2也为空则转移到stack2
            val := this.stack1[n1-1]
            this.stack1 = this.stack1[:n1-1]
            this.stack2 = append(this.stack2, val)
            n1 = len(this.stack1)
        }
    }
    n2 = len(this.stack2)
    if n2 == 0 {
        return -1
    }
    res := this.stack2[n2-1]
    this.stack2 = this.stack2[:n2-1]
    return res
}


/**
 * Your CQueue object will be instantiated and called as such:
 * obj := Constructor();
 * obj.AppendTail(value);
 * param_2 := obj.DeleteHead();
 */
```

二刷：

```go

// 思路：当第二个栈为空的时候，将第一个栈全部转移到第二个栈（让其逆序）
type CQueue struct {
    stack1 []int
    stack2 []int
}


func Constructor() CQueue {
    return CQueue{
        make([]int, 0),
        make([]int, 0),
    }
}

// 在尾部添加
func (this *CQueue) AppendTail(value int)  {
    this.stack1 = append(this.stack1, value)            // 直接添加在stack1
}

// 删除头节点
func (this *CQueue) DeleteHead() int {
    // 如果栈2的长度为0，那么就转移栈1的数，逆序
    n, m := len(this.stack1), len(this.stack2)
    if m == 0 {
        for n > 0 {
            // 转移
            v := this.stack1[n-1]
            this.stack1 = this.stack1[:n-1]
            this.stack2 = append(this.stack2, v)
            n = len(this.stack1)
        }
    }
    // 出队
    m = len(this.stack2)
    if m == 0 {
        // 如果即使移动过后还是为空，那么就没有元素返回-1
        return -1
    }
    res := this.stack2[m-1]
    this.stack2 = this.stack2[:m-1]
    return res
}


/**
 * Your CQueue object will be instantiated and called as such:
 * obj := Constructor();
 * obj.AppendTail(value);
 * param_2 := obj.DeleteHead();
 */
```

二刷

```go
type MyQueue struct {
    stack1 []int
    stack2 []int
}


func Constructor() MyQueue {
    return MyQueue{[]int{}, []int{}}
}


func (this *MyQueue) Push(x int)  {
    this.stack1 = append(this.stack1, x)
}


func (this *MyQueue) Pop() int {
    // 判断第二个栈是否有元素
    if len(this.stack2) == 0 {
        this.migration()
    }
    if len(this.stack2) == 0 {
        return -1
    }
    // 出栈
    ans := this.stack2[len(this.stack2)-1]
    this.stack2 = this.stack2[:len(this.stack2)-1]
    return ans
}

func (this *MyQueue) migration() {
    // 移动第一个栈元素到第二个栈
    for len(this.stack1) > 0 {
        val := this.stack1[len(this.stack1)-1]
        this.stack1 = this.stack1[:len(this.stack1)-1]
        this.stack2 = append(this.stack2, val)
    }
}


func (this *MyQueue) Peek() int {
    if len(this.stack2) == 0 {
        this.migration()
    }
    if len(this.stack2) == 0 {
        return -1
    }
    return this.stack2[len(this.stack2)-1]
}


func (this *MyQueue) Empty() bool {
    return len(this.stack1) == 0 && len(this.stack2) == 0
}


/**
 * Your MyQueue object will be instantiated and called as such:
 * obj := Constructor();
 * obj.Push(x);
 * param_2 := obj.Pop();
 * param_3 := obj.Peek();
 * param_4 := obj.Empty();
 */
```

