---
title: 删除链表中倒数第k个节点
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 删除链表中倒数第k个节点
#算法/链表/快慢指针
#算法/剑指offer

快慢指针法，快指针比慢指针的起始位置多k个，之后两个指针都走一步，直到快指针走到尾，慢指针的位置即为倒数第k个

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func getKthFromEnd(head *ListNode, k int) *ListNode {
    if head == nil {
        return nil
    }
    slow := head
    fast := head
    for k > 0 {
        if fast == nil {
            return nil
        }
        fast = fast.Next
        k--
    }
    for fast != nil {
        fast = fast.Next
        slow = slow.Next
    }
    return slow
}
```