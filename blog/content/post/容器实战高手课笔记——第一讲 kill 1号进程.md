---
title: 容器实战高手课笔记——第一讲 kill 1号进程
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["极客时间"]
tags: ["极客时间"]
---

# 容器实战高手课笔记——第一讲 kill 1号进程
#技术/极客时间/容器实战高手课

## 1号进程
如何理解 init 进程？
init 进程的意思并不难理解，使用容器的理想境界是一个容器只启动一个进程，但这在现实应用中有时是做不到的。
比如说，在一个容器中除了主进程之外，我们可能还会启动辅助进程，做监控或者 rotate logs；再比如说，我们需要把原来运行在虚拟机（VM）的程序移到容器里，这些原来跑在虚拟机上的程序本身就是多进程的。一旦我们启动了多个进程，那么容器里就会出现一个 pid 1，也就是我们常说的 1 号进程或者 init 进程，然后由这个进程创建出其他的子进程。
1 号进程是第一个用户态的进程，由它直接或者间接创建了 Namespace 中的其他进程

## Linux信号
我们运行 kill 命令，其实在 Linux 里就是发送一个信号
信号这个概念在很早期的 Unix 系统上就有了。它一般会从 1 开始编号，通常来说，信号编号是 1 到 31，这个编号在所有的 Unix 系统上都是一样的。在 Linux 上我们可以用 kill -l 来看这些信号的编号和名字
```sh
$ kill -l
 1) SIGHUP      2) SIGINT    3) SIGQUIT    4) SIGILL    5) SIGTRAP
 6) SIGABRT     7) SIGBUS    8) SIGFPE     9) SIGKILL  10) SIGUSR1
11) SIGSEGV    12) SIGUSR2  13) SIGPIPE   14) SIGALRM  15) SIGTERM
16) SIGSTKFLT  17) SIGCHLD  18) SIGCONT   19) SIGSTOP  20) SIGTSTP
21) SIGTTIN    22) SIGTTOU  23) SIGURG    24) SIGXCPU  25) SIGXFSZ
26) SIGVTALRM  27) SIGPROF  28) SIGWINCH  29) SIGIO    30) SIGPWR
31) SIGSYS
```

用一句话来概括，信号（Signal）其实就是 Linux 进程收到的一个通知。这些通知产生的源头有很多种，通知的类型也有很多种。比如下面这几个典型的场景：
- 如果我们按下键盘“Ctrl+C”，当前运行的进程就会收到一个信号 SIGINT 而退出；
- 如果我们的代码写得有问题，导致内存访问出错了，当前的进程就会收到另一个信号 SIGSEGV；
- 我们也可以通过命令 kill ，直接向一个进程发送一个信号，缺省情况下不指定信号的类型，那么这个信号就是 SIGTERM。
- 也可以指定信号类型，比如命令 “kill -9 “, 这里的 9，就是编号为 9 的信号，SIGKILL 信号。

进程在收到信号后，就会去做相应的处理。怎么处理呢？对于每一个信号，进程对它的处理都有下面三个选择。
- 第一个选择是忽略（Ignore），就是对这个信号不做任何处理，但是有两个信号例外，对于 SIGKILL 和 SIGSTOP 这个两个信号，进程是不能忽略的。
这是因为它们的主要作用是为 Linux kernel 和超级用户提供删除任意进程的特权。
第二个选择，就是捕获（Catch），这个是指让用户进程可以注册自己针对这个信号的 handler。对于捕获，SIGKILL 和 SIGSTOP 这两个信号也同样例外，这两个信号不能有用户自己的处理代码，只能执行系统的缺省行为。
- 还有一个选择是缺省行为（Default），Linux 为每个信号都定义了一个缺省的行为，你可以在 Linux 系统中运行 man 7 signal来查看每个信号的缺省行为。对于大部分的信号而言，应用程序不需要注册自己的 handler，使用系统缺省定义行为就可以了。

SIGTERM（15），这个信号是 Linux 命令 kill 缺省发出的。前面例子里的命令 kill 1 ，就是通过 kill 向 1 号进程发送一个信号，在没有别的参数时，这个信号类型就默认为 SIGTERM。SIGTERM 这个信号是可以被捕获的，这里的“捕获”指的就是用户进程可以为这个信号注册自己的 handler，而这个 handler，我们后面会看到，它可以处理进程的 graceful-shutdown 问题

我们再来了解一下 SIGKILL (9)，这个信号是 Linux 里两个特权信号之一。什么是特权信号呢？前面我们已经提到过了，特权信号就是 Linux 为 kernel 和超级用户去删除任意进程所保留的，不能被忽略也不能被捕获。那么进程一旦收到 SIGKILL，就要退出。在前面的例子里，我们运行的命令 kill -9 1 里的参数“-9”，其实就是指发送编号为 9 的这个 SIGKILL 信号给 1 号进程。


Linux 内核针对每个 Nnamespace 里的 init 进程，把只有 default handler 的信号都给忽略了

如果我们自己注册了信号的 handler（应用程序注册信号 handler 被称作”Catch the Signal”），那么这个信号 handler 就不再是 SIG_DFL 。即使是 init 进程在接收到 SIGTERM 之后也是可以退出的

不过，由于 SIGKILL 是一个特例，因为 SIGKILL 是不允许被注册用户 handler 的（还有一个不允许注册用户 handler 的信号是 SIGSTOP），那么它只有 SIG_DFL handler。所以 init 进程是永远不能被 SIGKILL 所杀，但是可以被 SIGTERM 杀死

## 总结
1. kill -9 1 在容器中是不工作的，内核阻止了 1 号进程对 SIGKILL 特权信号的响应。
2. kill 1 分两种情况
	1. 如果 1 号进程没有注册 SIGTERM 的 handler，那么对 SIGTERM 信号也不响应
	2. 如果注册了 handler，那么就可以响应 SIGTERM 信号。

如果有一个 C 语言的 init 进程，它没有注册任何信号的 handler。如果我们从 Host Namespace 向它发送 SIGTERM，会发生什么情况呢？

1. kill -9 可以杀掉，因为从Host Namespace的角度来看，它并不是init进程，因此SIGKILL信号不会被忽略，必须执行缺省的handler，被杀掉
2. kill 不可以杀掉，因为它没有注册handler，无法接收SIGTERM信号 