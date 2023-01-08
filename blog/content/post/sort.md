---
title: sort
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["golang"]
tags: ["golang"]
---

# sort
#go

```go
ackage main

import (
	"fmt"
	"sort"
)

func main() {
	sortInt()
	sortIntOrigin()
	sortIntReverseOrigin()
	judgeSorted()
}

type IntSlice []int

func (s IntSlice) Len() int {
	return len(s)
}

func (s IntSlice) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}

func (s IntSlice) Less(i, j int) bool {
	return s[i] < s[j]
}

func sortInt() {
	s := []int{5,6,3,2,1,4,8,9,7}
	sort.Sort(IntSlice(s))
	fmt.Println(s)
}

//sort包提供了原生的IntSlice，可以直接对数组排序，不用再自己定义
//slice类型并实现其Len Less Swap接口了
func sortIntOrigin() {
	s := sort.IntSlice{5,6,7,1,2,3,4,8,9}
	sort.Sort(s)
	fmt.Println(s)
}

//sort包提供了逆序排序的功能，首先需要用Reverse包装
//原理是将待排序结构作为内嵌字段封装在了sort包内置的reverse结构中
//因为reverse将外部传入的排序Interface作为其内嵌字段
//所以它也就继承了其所有方法，它也是一个Interface
//但是，它将Interface的Less方法比大小时交换了
//这样在真实排序时调用此方法即得到逆转的结果了
func sortIntReverseOrigin() {
	s := sort.IntSlice{5,6,7,1,2,3,4,8,9}
	sort.Sort(sort.Reverse(s))
	fmt.Println(s)
}

//判断是否排好序
func judgeSorted() {
	s := sort.IntSlice{1,2,3,4,5,6,7,8,9}
	fmt.Println(sort.IsSorted(s))
}


```