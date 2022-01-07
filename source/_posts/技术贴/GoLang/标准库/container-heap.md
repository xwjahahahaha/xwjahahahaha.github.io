---
title: container/heap
tags:
  - golang
categories:
  - technical
  - golang
toc: true
declare: true
date: 2021-04-28 10:11:27
---

# container/heap

### 基本信息

堆的go源码实现

heap库源码位置:`$GOROOT/src/container/heap/heap.go`

学习资源:https://blog.csdn.net/weixin_34104341/article/details/91929146

<!-- more -->

### 使用

一般需要实现五个方法: Len、Less、Swap、Push、Pop

heap包中提供了几个最基本的堆操作函数，包括Init，Fix，Push，Pop和Remove （其中up, down函数为非导出函数）。这些函数都通过调用前面实现接口里的方法，对堆进行操作。

#### Init

```go
func Init(h Interface)
```

一个堆在使用任何堆操作之前应先初始化。接受参数为实现了heap.Interface接口的对象。

#### Fix

```go
func Fix(h Interface, i int)
```

在修改第i个元素后，调用本函数修复堆，比删除第i个元素后插入新元素更有效率。 

#### Push&Pop

Push和Pop是一对标准堆操作，Push向堆添加一个新元素，Pop弹出并返回堆顶元素，而在push和pop操作不会破坏堆的结构

```go
func Push(h Interface, x interface{})
func Pop(h Interface) interface{} 
```

#### **Remove**

删除堆中的第i个元素，并保持堆的约束性

```go
func Remove(h Interface, i int) interface{}
```

### 案例

#### [347. 前 K 个高频元素](https://leetcode-cn.com/problems/top-k-frequent-elements/)

```go
// 堆/优先队列
// 对于统计数组进行堆的topk
// 构建一个小顶堆, 当堆规模 < k 时, 直接添加元素,保持堆属性
// 当堆规模 == k时, 当堆顶 > 新值时, 舍弃该值 (因为整个堆都大于该值即其是第k+1大的数)
// 当堆规模 == k时, 当堆顶 <= 新值时, 删除堆顶值,插入当前值,调整堆属性 (更新这K规模的堆,堆顶最小值可能由此抬高)
// 时间复杂度O(NlogK), 每次堆操作为最大为logK, 最坏N次
func topKFrequent(nums []int, k int) (ans []int) {
    // 统计次数数组
    countMap := make(map[int]int)
    for _, num := range nums {
        countMap[num] ++
    }
    // 初始化堆
    myHeap := &IHeap{}
    heap.Init(myHeap)
    // 遍历次数数组入堆
    for num, count := range countMap {
        heap.Push(myHeap, [2]int{num, count})
        // 大于K个就弹出
        if myHeap.Len() > k {
            heap.Pop(myHeap)
        }
    }
    // 一直弹出,整合结果
    ans = make([]int, k)
    for i:=0; i<k; i++ {
        // 注意倒序、接口断言
        ans[k-i-1] = heap.Pop(myHeap).([2]int)[0] 
    } 
    return 
}

type IHeap [][2]int                 // 二元组,(num, count)
// 实现需要的方法
func (h IHeap)Len() int {
    return len(h)
}
func (h IHeap)Less(i, j int) bool {
    return h[i][1] < h[j][1]        // 小顶堆
}
func (h IHeap)Swap(i, j int) {
    h[i], h[j] = h[j], h[i]
}
func (h *IHeap)Push(x interface{}) {
    *h = append(*h, x.([2]int))     // 断言类型后添加
}
func (h *IHeap)Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    // 弹出
    *h = old[:n-1]
    return x
}
```

#### 官方案例intHeap: 最小堆

```go
// An IntHeap is a min-heap of ints.
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x interface{}) {
	// Push and Pop use pointer receivers because they modify the slice's length,
	// not just its contents.
	*h = append(*h, x.(int))
}

func (h *IntHeap) Pop() interface{} {
	old := *h
	n := len(old)
	x := old[n-1]
	*h = old[0 : n-1]
	return x
}

// This example inserts several ints into an IntHeap, checks the minimum,
// and removes them in order of priority.
func Example_intHeap() {
	h := &IntHeap{2, 1, 5}
	heap.Init(h)
	heap.Push(h, 3)
	fmt.Printf("minimum: %d\n", (*h)[0])
	for h.Len() > 0 {
		fmt.Printf("%d ", heap.Pop(h))
	}
	// Output:
	// minimum: 1
	// 1 2 3 5
}
```

#### 优先队列案例

```go
// An Item is something we manage in a priority queue.
type Item struct {
	value    string // The value of the item; arbitrary.
	priority int    // The priority of the item in the queue.
	// The index is needed by update and is maintained by the heap.Interface methods.
  // id参数是用于特定更新堆中的某个元素的优先级等值,并在方法中维护.如果没有此需求可以不实现
	index int // The index of the item in the heap.
}

// A PriorityQueue implements heap.Interface and holds Items.
type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }

func (pq PriorityQueue) Less(i, j int) bool {
	// We want Pop to give us the highest, not lowest, priority so we use greater than here.
  // 这里是大顶堆,一般优先级按大顶堆
	return pq[i].priority > pq[j].priority
}

func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	// 注意这里需要确定index
	pq[i].index = i
	pq[j].index = j
}

func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item)
	item.index = n
	*pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	old[n-1] = nil  // avoid memory leak  避免内存泄漏
	item.index = -1 // for safety
	*pq = old[0 : n-1]
	return item
}

// update modifies the priority and value of an Item in the queue.
// 更新堆中的元素优先级，并重新调整堆属性
func (pq *PriorityQueue) update(item *Item, value string, priority int) {
	item.value = value
	item.priority = priority
	heap.Fix(pq, item.index)
}

// This example creates a PriorityQueue with some items, adds and manipulates an item,
// and then removes the items in priority order.
func Example_priorityQueue() {
	// Some items and their priorities.
	items := map[string]int{
		"banana": 3, "apple": 2, "pear": 4,
	}

	// Create a priority queue, put the items in it, and
	// establish the priority queue (heap) invariants.
	pq := make(PriorityQueue, len(items))
	i := 0
	for value, priority := range items {
		pq[i] = &Item{
			value:    value,
			priority: priority,
			index:    i,
		}
		i++
	}
	heap.Init(&pq)

	// Insert a new item and then modify its priority.
	item := &Item{
		value:    "orange",
		priority: 1,
	}
	heap.Push(&pq, item)
	pq.update(item, item.value, 5)

	// Take the items out; they arrive in decreasing priority order.
	for pq.Len() > 0 {
		item := heap.Pop(&pq).(*Item)
		fmt.Printf("%.2d:%s ", item.priority, item.value)
	}
	// Output:
	// 05:orange 04:pear 03:banana 02:apple
}
```





