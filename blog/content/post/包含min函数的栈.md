---
title: 包含min函数的栈
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 包含min函数的栈
#算法/设计
#算法/栈
[力扣](https://leetcode-cn.com/problems/bao-han-minhan-shu-de-zhan-lcof/)

核心思路：
需要两个栈结构：
- 一个是普通的栈，用来做正常的栈压入、弹出等操作
- 一个是有序的栈，它只插入比自己栈顶元素小的元素（注意等于栈顶也需要插入，因为若相同元素被插入两次，如果不同步插入sorted栈两次，那么getMin多次后，sorted栈会少元素），因为插入比栈顶的元素大的元素没有意义，getMin不会用到这种元素，弹出时因为大的元素是后进来的，它会先被弹出。这样可以保证这个栈是有序的。弹出时，主栈直接弹出，若主栈弹出元素等于有序栈的栈顶元素，则把有序栈的栈顶元素也弹出。这样可以保证有序栈的栈顶元素总是当前正常栈中最小的，因为比栈顶大的元素的插入和弹出都没有经过这个栈。

```go
type MinStack struct {
    stack []int
    orderedStack []int
}


/** initialize your data structure here. */
func Constructor() MinStack {
    return MinStack{stack:[]int{}, orderedStack:[]int{}}
}


func (this *MinStack) Push(x int)  {
    this.stack = append(this.stack, x)
    if len(this.orderedStack) == 0 || this.orderedStack[len(this.orderedStack) - 1] >= x {
        this.orderedStack = append(this.orderedStack, x)
    }
}


func (this *MinStack) Pop()  {
    if len(this.stack) > 0 {
        poped := this.stack[len(this.stack) - 1]
        this.stack = this.stack[:len(this.stack) - 1]
        if len(this.orderedStack) > 0 {
            if this.orderedStack[len(this.orderedStack) - 1] == poped {
                this.orderedStack = this.orderedStack[:len(this.orderedStack) - 1]
            }
        }
    }
}


func (this *MinStack) Top() int {
    if len(this.stack) > 0 {
        return this.stack[len(this.stack) - 1]
    }
    return 0
}


func (this *MinStack) Min() int {
    if len(this.orderedStack) > 0 {
        return this.orderedStack[len(this.orderedStack) - 1]
    }
    return 0
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