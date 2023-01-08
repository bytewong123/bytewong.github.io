---
title: 数据流中的第k大元素
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 数据流中的第k大元素
#算法/堆

```
设计一个找到数据流中第 k 大元素的类（class）。注意是排序后的第 k 大元素，不是第 k 个不同的元素。

请实现 KthLargest 类：

KthLargest(int k, int[] nums) 使用整数 k 和整数流 nums 初始化对象。
int add(int val) 将 val 插入数据流 nums 后，返回当前数据流中第 k 大的元素。
 

示例：

输入：
["KthLargest", "add", "add", "add", "add", "add"]
[[3, [4, 5, 8, 2]], [3], [5], [10], [9], [4]]
输出：
[null, 4, 5, 5, 8, 8]

解释：
KthLargest kthLargest = new KthLargest(3, [4, 5, 8, 2]);
kthLargest.add(3);   // return 4
kthLargest.add(5);   // return 5
kthLargest.add(10);  // return 5
kthLargest.add(9);   // return 8
kthLargest.add(4);   // return 8
```

本题跟topk是一样的思路。因为需要返回第k大的元素，即堆中只开辟k大小的空间，并且是一个小顶堆。这样的话可以保证比堆顶小的元素进不来，且堆顶一直是倒数第k小的，因为堆中的其他元素都比它大

```go
type KthLargest struct {
    heap []int
    size int
}


func Constructor(k int, nums []int) KthLargest {
    c := KthLargest{
        heap : make([]int, 0, k),
        size : k,
    }
    for _, num := range nums {
        c.push(num)
    }
    return c
}


func (this *KthLargest) Add(val int) int {
    this.push(val)
    if len(this.heap) > 0 {
        return this.heap[0]
    }
    return -1
}

func (this *KthLargest) push(val int) {
    if len(this.heap) < this.size {
        this.pushInHeap(val)
    } else {
        this.popAndAdjustHeap(val)
    }
}

func (this *KthLargest) pushInHeap(val int) {
    this.heap = append(this.heap, val)
    up(this.heap, len(this.heap) - 1)
}

func up(nums []int, cur int) {
    parent := (cur - 1) / 2
    if parent < 0 {
        return
    }
    if nums[parent] > nums[cur] {
        nums[parent], nums[cur] = nums[cur], nums[parent]
        up(nums, parent)
    }
}

func (this *KthLargest) popAndAdjustHeap(val int) {
    if val < this.heap[0] {
        return
    }
    this.heap[0] = val
    down(this.heap, 0)
}

func down(nums []int, cur int) {
    left := cur * 2 + 1
    right := cur * 2 + 2
    smaller := cur
    if left < len(nums) && nums[smaller] > nums[left] {
        smaller = left
    }
    if right < len(nums) && nums[smaller] > nums[right] {
        smaller = right
    }
    if smaller != cur {
        nums[cur], nums[smaller] = nums[smaller], nums[cur]
        down(nums, smaller)
    }
}

/**
 * Your KthLargest object will be instantiated and called as such:
 * obj := Constructor(k, nums);
 * param_1 := obj.Add(val);
 */
```