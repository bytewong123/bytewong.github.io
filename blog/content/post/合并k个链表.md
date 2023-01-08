---
title: 合并k个链表
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 合并k个链表
#算法/链表
#算法/合并链表
#算法/堆

本题用堆排序的方式。首先把k个链表的头节点都入小顶堆。然后开始不断弹出堆，弹出的总是最小的那一个，然后结果指针指向这个元素。若弹出元素的后置节点不为空，再继续入堆。这样每次弹出能保证是按从小到大的顺序弹出。当堆为空了，则结束。

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeKLists(lists []*ListNode) *ListNode {
    hat := new(ListNode)
    cur := hat
    heap := []*ListNode{}
    for _, list := range lists {
        if list != nil {
            heap = append(heap, list)
            up(heap, len(heap) - 1)
        }
    }
    fmt.Println(heap)
    for len(heap) != 0 {
        poped := popHeap(&heap)
        cur.Next = poped
        cur = poped
        if poped.Next != nil {
            heap = append(heap, poped.Next)
            up(heap, len(heap) - 1)
        }
    }
    return hat.Next
}

func pushInHeap(heap []*ListNode, node *ListNode) {
    up(heap, len(heap) - 1)
}

func popHeap(heap *[]*ListNode) *ListNode {
    var poped *ListNode
    if len(*heap) < 1 {
        return nil
    }
    if len(*heap) == 1 {
        poped = (*heap)[0]
        *heap = (*heap)[1:]
        return poped
    } else {
        poped = (*heap)[0]
        (*heap)[0] = (*heap)[len(*heap) - 1]
        *heap = (*heap)[:len(*heap) - 1]
    }
    down(*heap, 0)
    return poped
}

func up(heap []*ListNode, cur int) {
    if cur <= 0 {
        return
    }
    parent := (cur - 1) / 2
    if heap[parent].Val > heap[cur].Val {
        heap[parent], heap[cur] = heap[cur], heap[parent]
        up(heap, parent)
    }
}

func down(heap []*ListNode, cur int) {
    left, right := cur * 2 + 1, cur * 2 + 2
    smaller := cur
    if left < len(heap) && heap[left].Val < heap[smaller].Val {
        smaller = left
    }
    if right < len(heap) && heap[right].Val < heap[smaller].Val {
        smaller = right
    }
    if smaller != cur {
        heap[smaller], heap[cur] = heap[cur], heap[smaller]
        down(heap, smaller)
    }
}
```