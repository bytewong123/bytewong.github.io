---
title: 买卖股票最多只能买卖k次的最大利润
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# 买卖股票最多只能买卖k次的最大利润
#算法/dp/股票问题

本题跟买卖两次类似，但是两次变为了k次，需要将两次转变为一般的情况，但是思路是相同的

对于交易2次，有如下状态
1. 0，不作为
2. 1，买了1次
3. 2，买了1次卖了1次
4. 3，买了1次，卖了1次，又买了1次
5. 4，买了2次，卖了2次

可以发现规律：
1. k次买卖，有2 * k + 1种状态
2. 除了0的偶数状态，会在前一个状态的基础上买股票
3. 奇数状态，会在前一个状态的基础上卖股票
因此，
1. 当处于一个偶数状态时，当前日期的最大收益应该是前一天相同状态的最大收益与前一天的前一个状态的最大收益再买入今日价格的股票的最大收益的最大值
2. 当处于一个奇数状态时，当前日期的最大收益应该是前一天相同状态的最大收益与前一天的前一个状态的最大收益再卖出今日价格的股票的最大收益的最大值

注意状态的临界情况的处理，可以边举例子边写，例如k=2的各个情况


```go
func maxProfit(k int, prices []int) int {
    if len(prices) == 0 {
        return 0
    }
    dp := make([][]int, len(prices))
    for i := 0; i < len(dp); i++ {
        dp[i] = make([]int, 2 * k + 1)
    }
    dp[0][0] = 0
    dp[0][1] = -prices[0]
    for i := 2; i < 2 * k + 1; i++ {
        dp[0][i] = -(1<<31)
    }
    for i := 1; i < len(prices); i++ {
        for j := 0; j < 2 * k; j += 2 {
            dp[i][j + 1] = max(dp[i - 1][j + 1], dp[i - 1][j] - prices[i])
            dp[i][j + 2] = max(dp[i - 1][j + 2], dp[i - 1][j + 1] + prices[i])
        } 
    }
    fmt.Println(dp)
    return dp[len(prices) - 1][2 * k]
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```