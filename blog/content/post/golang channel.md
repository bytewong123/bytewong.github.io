---
title: golang channel
date: 2023-01-08T18:06:57+08:00
draft: false
categories: [""]
tags: [""]
---

# golang channel
#go

## <-读取
- `<-c` 读一个管道，若管道内无数据，将会一直阻塞；
- 若管道关闭，继续读这个关闭的管道会读取到该数据类型的零值
- 若使用`v, ok := <-c`来读取，管道未关闭，有数据时v为读到的数据，ok为true；无数据了，会一直阻塞；若管道关闭了，会读到零值，ok为false。

## range读取
- 若管道不关闭，管道内无数据了，将会一直阻塞在管道，直到管道关闭
- 管道关闭了，会退出循环

## 缓存channel的作用：
缓存的使用，可以尽量避免阻塞，提供应用的性能

## 往一个关闭的channel写数据会引发panic

## channel定义
```
chan T          // 可以接收和发送类型为 T 的数据
chan<- T  	  // 只可以用来发送 T 类型的数据
<-chan T        // 只可以用来接收 T 类型的数据
```
如果指定了channel是只可发送或只可接收的channel，但是却做了与指定的channel数据流向不一致的操作，会在编译期间报错。相当于是一种检查机制。
例如：
```
invalid operation: <-inchan (receive from send-only type chan<- int)
```

channel关闭之后，仍然可以从channel中读取剩余的数据，直到数据全部读取完成。
多说一点，对于一个关闭的channel，如果继续向channel发送数据，会引起panic

奇技淫巧
用一个管道统一处理所有请求；请求中带上该请求的recv channel；处理完毕后向请求的recv channel发送结果；请求只需监听该recv channel即可。详见zk/conn.go，queueRequest