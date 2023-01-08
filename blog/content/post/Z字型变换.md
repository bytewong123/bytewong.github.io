---
title: Z字型变换
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["算法"]
tags: ["算法"]
---

# Z字型变换
#算法/数组
#算法/仍然不会

```
将一个给定字符串 s 根据给定的行数 numRows ，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 "PAYPALISHIRING" 行数为 3 时，排列如下：

P   A   H   N
A P L S I I G
Y   I   R
之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："PAHNAPLSIIGYIR"。

请你实现这个将字符串进行指定行数变换的函数：

string convert(string s, int numRows);
```

本题的关键在于使用一个flag。创建和行数相同的n个byte数组。原字符串是Z字型的字符串，因此模拟这个字符串的读取顺序，将其加入到对应的行上。如果是第一行，那么应该往下走，flag为1，下一次加入的字符应该是当前行+1即第二行，以此类推。当到达最后一行，flag翻转为-1，代表需要向上走了，因此下一个字符放置的位置应该是当前行-1

```go
func convert(s string, numRows int) string {
    if numRows == 1 {
        return s
    }
    bs := make([][]byte, numRows)
    flag := -1
    idx := 0
    for i := 0; i < len(s); i++ {
        bs[idx] = append(bs[idx], s[i])
        if idx == 0 || idx == numRows - 1 {
            flag = -flag
        }
        idx += flag
    }
    ans := []byte{}
    for i := 0; i < numRows; i++ {
        ans = append(ans, bs[i]...)
    }
    return string(ans)
}
```