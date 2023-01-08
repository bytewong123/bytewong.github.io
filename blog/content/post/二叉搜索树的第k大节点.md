---
title: 二叉搜索树的第k大节点
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 二叉搜索树的第k大节点
#算法/二叉树

## 链接
[力扣](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-di-kda-jie-dian-lcof/)

## 题目
```
给定一棵二叉搜索树，请找出其中第 k 大的节点的值。

 

示例 1:

输入: root = [3,1,4,null,2], k = 1
   3
  / \
 1   4
  \
   2
输出: 4
示例 2:

输入: root = [5,3,6,2,4,null,null,1], k = 3
       5
      / \
     3   6
    / \
   2   4
  /
 1
输出: 4
 

限制：

1 ≤ k ≤ 二叉搜索树元素个数
```

## 思路
二叉搜索树 左根右 顺序递归为升序序列
二叉搜索树 右根左 顺序递归为降序序列

因此求第k大用**右根左**递归；求第k小用**左根右**递归

## 答案
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func kthLargest(root *TreeNode, k int) int {
    if root == nil {
        return 0
    }
    ans := 0
    cur := 0
    dfs(root, &cur, &ans, k)
    return ans
}

func dfs(root *TreeNode, cur, ans *int, k int) {
    if root == nil {
        return
    }
    dfs(root.Right, cur, ans, k)
    *cur++
    if *cur == k {
        *ans = root.Val
    }
    dfs(root.Left, cur, ans, k)
}

```


