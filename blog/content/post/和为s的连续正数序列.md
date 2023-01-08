---
title: 和为s的连续正数序列
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 和为s的连续正数序列
#算法/剑指offer
#算法/滑动窗口
#算法/仍然不会

## 题目
```
输入一个正整数 target ，输出所有和为 target 的连续正整数序列（至少含有两个数）。

序列内的数字由小到大排列，不同序列按照首个数字从小到大排列。

 

示例 1：

输入：target = 9
输出：[[2,3,4],[4,5]]
示例 2：

输入：target = 15
输出：[[1,2,3,4,5],[4,5,6],[7,8]]
 

限制：

1 <= target <= 10^5
```

## 思路
滑动窗口，如果大于target了，那么舍弃掉左指针，窗口往右移动：减去左指针曾经加过的值，左指针+1，然后继续右扩指针

## 答案
```go
func findContinuousSequence(target int) [][]int {
    if target < 2 {
        return nil
    }
    seq := make([]int, target - 1) 
    for i := 0; i < len(seq); i++ {
        seq[i] = i + 1
    }
    left, right := 0, 1
    sum := seq[left]
    ans := make([][]int, 0)
    for right < len(seq) {
        if sum > target {
            sum -= seq[left]
            left++
            continue
        }  
        if sum == target {
            ans = append(ans, makeSeq(left, right - 1))
            sum -= seq[left]
            left++
            continue
        }
        sum += seq[right]
        right++
    }
    return ans
}

func makeSeq(left, right int) []int {
    ans := make([]int, right - left + 1)
    for i := 0; i < len(ans); i++ {
        ans[i] = left+1
        left++
    }
    return ans
}

```