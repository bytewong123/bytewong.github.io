---
title: 翻转链表2（翻转第m到第n个链表节点之间的元素）
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 翻转链表2（翻转第m到第n个链表节点之间的元素）
#算法/链表/翻转链表

思路：
1. 遍历，找到m和n对应的节点以及m前一个节点和n后一个节点，翻转这段链表需要这四个节点
2. 翻转这四个节点即可，注意这四个节点边界的指向关系，画图能比较明显地看出来
3. 如果m和n是一个节点，不用翻转

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseBetween(head *ListNode, m int, n int) *ListNode {
    if head == nil {
        return nil
    }
    hat := new(ListNode)
    hat.Next = head
    pre, startPre := hat, hat
    cur, start, end := hat.Next, hat.Next, hat.Next
    endAfter := cur.Next
    count := 1
    for {
        if count == m {
            start = cur
            startPre = pre
        }
        if count == n {
            end = cur
            endAfter = end.Next
            break
        }
        pre = pre.Next
        cur = cur.Next
        count++
    }
    reverse(startPre, start, endAfter, end)
    return hat.Next
}

func reverse(startPre, start, endAfter, end *ListNode) {
    //fmt.Println(startPre, start, endAfter, end)
    if start == end {
        return
    }
    pre := start
    cur := start.Next
    for {
        next := cur.Next
        cur.Next = pre
        if cur == end {
            break
        }
        pre = cur
        cur = next
    }
    start.Next = endAfter
    startPre.Next = end
}
```