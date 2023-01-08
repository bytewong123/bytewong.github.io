---
title: 最小的k个数
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 最小的k个数
#算法/堆
#算法/剑指offer

与topk题目相同。最小的k个数，构造大顶堆，每次有比堆顶元素小的元素加入，则从堆中把堆顶元素移出，将这个加入的元素放入到堆顶中。相当于一个优先级队列，每次把最大的元素弹出去，这样队列里面维护的就是最小的一批数

```go
func getLeastNumbers(arr []int, k int) []int {
    if k == 0 {
        return nil
    }
    heap := make([]int, 0)
    for _, num := range arr {
        if len(heap) < k {
            pushInHeap(&heap, num)
        } else {
            if heap[0] > num {
                popAndAdjustHeap(heap, num)
            }
        }
    }
    return heap
}

func pushInHeap(heap *[]int, num int) {
    *heap = append(*heap, num)
    up(*heap, len(*heap) - 1)
}

func up(heap []int, cur int) {
    parent := (cur - 1) / 2
    if heap[parent] < heap[cur] {
        swap(heap, cur, parent)
        up(heap, parent)
    } else {
        return
    }
}

func swap(heap []int, a int, b int) {
    heap[a], heap[b] = heap[b], heap[a]
}

func popAndAdjustHeap(heap []int, num int) {
    heap[0] = num
    down(heap, 0)
}

func down(heap []int, cur int) {
    left := cur * 2 + 1
    right := cur * 2 + 2
    biggest := cur
    if left < len(heap) && heap[left] > heap[biggest] {
        biggest = left
    }
    if right < len(heap) && heap[right] > heap[biggest] {
        biggest = right
    }
    if biggest != cur {
        swap(heap, biggest, cur)
        down(heap, biggest)
    }
}
```