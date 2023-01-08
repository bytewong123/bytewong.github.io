---
title: io模型
date: 2023-01-08T18:06:57+08:00
draft: false
categories: ["操作系统"]
tags: ["操作系统"]
---

# io模型
#技术/io模型

## 同步io
- 阻塞io
如果内核中数据没有准备好，阻塞用户进程直到数据准备完成
- 非阻塞io
如果内核中数据没有准备好，不会阻塞用户进程，而是立刻返回一个error
典型应用：socket设置nonblock
缺点：进程轮询调用，消耗cpu资源
- io多路复用
	1. 多个进程的io可以注册到一个复用器（select）上
	2. 当用户调用该select，select会监听所有注册进来的io
	3. 如果select所有监听的io在内核缓冲区都没有可读取的数据，select调用进程会被阻塞
	4. 任一io有数据，select就会返回
	5. 复制数据到内存缓冲区期间，进程阻塞
	6. IO复用模型中，对于每一个socket，一般都设置成为非阻塞。但是整个用户的进程其实是一直被阻塞的，只不过进程是被select这个函数阻塞、而不是被socket IO给阻塞
	
	优缺点：
		1. 事实上比阻塞io性能更差一些，因为需要有两次系统调用select,recvfrom，阻塞io只需要一次recvfrom
		2. 优势在于可以同时处理多个连接
		3. 专一进程解决多个进程IO的阻塞问题，适用高并发服务应用开发，一个进程/线程响应多个请求
	
	典型应用：
	Java NIO、Nginx（epoll、poll、select）

- 信号驱动io
	1. 进程预先告知内核、向内核注册一个信号处理函数
	2. 用户进程返回不阻塞，当内核数据就绪时会发送一个信号给进程，用户进程便在信号处理函数中调用IO读取数据
	3. IO内核拷贝到用户进程的过程还是阻塞的，信号驱动式IO并没有实现真正的异步，因为通知到用户进程之后，依然是由进程来完成IO操作

	优缺点：
	半异步，并且实际中并不常用
	回调机制

## 异步io
- 异步io
用户进程只需要发起一个IO操作然后立即返回，等IO操作真正的完成以后，应用程序会得到IO操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的IO读写操作，因为真正的IO读取或者写入操作已经由内核完成了


## Reactor模式和Proactor模式
**Reactor模式应用于同步I/O的场景。我们分别以读操作和写操作为例来看看Reactor中的具体步骤：**
读取操作：
1. 应用程序注册读就绪事件和相关联的事件处理器
2. 事件分离器等待事件的发生
3. 当发生读就绪事件的时候，事件分离器调用第一步注册的事件处理器
4. 事件处理器首先执行实际的读取操作，然后根据读取到的内容进行进一步的处理
写入操作类似于读取操作，只不过第一步注册的是写就绪事件。

**Proactor模式中读取操作和写入操作的过程：**
读取操作：
1. 应用程序初始化一个异步读取操作，然后注册相应的事件处理器，此时事件处理器不关注读取就绪事件，而是关注读取完成事件，这是区别于Reactor的关键。
2. 事件分离器等待读取操作完成事件
3. 在事件分离器等待读取操作完成的时候，操作系统调用内核线程完成读取操作（异步IO都是操作系统负责将数据读写到应用传递进来的缓冲区供应用程序操作，操作系统扮演了重要角色），并将读取的内容放入用户传递过来的缓存区中。这也是区别于Reactor的一点，Proactor中，应用程序需要传递缓存区。
4. 事件分离器捕获到读取完成事件后，激活应用程序注册的事件处理器，事件处理器直接从缓存区读取数据，而不需要进行实际的读取操作。

从上面可以看出，Reactor和Proactor模式的主要区别就是真正的读取和写入操作是有谁来完成的，Reactor中需要应用程序自己读取或者写入数据，而Proactor模式中，应用程序不需要进行实际的读写过程，它只需要从缓存区读取或者写入即可，操作系统会读取缓存区或者写入缓存区到真正的IO设备.

## 几个区别
### 同步io和异步io的区别
同异步IO的根本区别在于、同步IO主动的调用recvfrom来将数据拷贝到用户内存、而异步则完全不同、它就像是用户进程将整个IO操作交给了他人（内核）完成、然后他人做完后发信号通知、在此期间、用户进程不需要去检查IO操作的状态、也不需要主动的去拷贝数据

### 阻塞io和非阻塞io的区别
阻塞和非阻塞关注的是进程在等待调用结果时的状态、阻塞是指调用结果返回之前、当前进程会被挂起、调用进程只有在得到结果才会返回；非阻塞调用指不能立刻得到结果、该调用不会阻塞当前进程。

### 同步和异步的区别
- 当一个同步调用发出去后，调用者要一直等待调用结果的通知后，才能进行后续的执行。
- 当一个异步调用发出去后，调用者不能立即得到调用结果的返回
	- 主动轮询异步调用的结果
	- 被调用方通过callback来通知调用方调用结果

## io多路复用详解
### select
函数原型：
```
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

- nfds，bitmap的最大长度
监控的所有文件描述符中，序号最大的一个描述符n的值+1，以便开辟n+1长度的bitmap，每一个位对应第0~n个文件描述符，若监听了则将其置为1，未监听将其置为0，这样每一位就代表监听的描述符。使用这个长度，主要是select函数通过这个长度去截断传入的bitmap
- readfds，读描述符集合
若select发现某个描述符有数据了，那么给这个bitmap对应的位置1
- writefds，写描述符集合
- exceptfds，异常描述符集合
- timeout，阻塞时间

缺点：
- bitmap的限制为1024
- 每次调用select，会把bitmap从用户态拷贝到内核态
- 每次读取bitmap结果，会遍历去查看是否有数据，复杂度为O(n)
- bitmap不可重用，因为每次select函数返回后就将其置1了，改变了内容，每次调用select需要新建bitmap


### poll
函数原型：
```
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
```

- pollfd数组
	- pollfd
		- fd
		- events：期望的监听事件
		- revents：poll返回时给该描述符置的事件
- 监听的fd数量
- 超时时间

优点：
1. fd可重用，因为每次poll函数返回了会改变pollfd.revents的结果，此时判断该结果，并做业务逻辑后，将revents置回0还原即可，下次还能继续使用该pollfd
2. pollfds是个数组，容量不只限于1024

### epoll
涉及三个函数

- epoll_create
创建一个eventpoll对象
	- rb_root，红黑树，存放需要监听的socket fd列表。在epollctl中往红黑树添加、修改、删除socket fd会很快
	- rdlist，双向链表，保存了事件已经到来的socket fd就绪列表
创建完成后，返回epollfd

- epoll_ctl
根据epollfd，给eventpoll对象增删改查具体的需要监听的描述符以及监听事件
函数原型：
```
int epoll_ctl(int epfd, int op, int fd, struct event *event);
```

	- epfd为eventpoll对象的fd，即epollfd
	- op为此次要对socket fd进行关联到epollfd的何种操作
	- fd是这次要操作的socket fd
	- event，告诉内核该socket需要监听的事件。添加到eventpoll中的事件都会与设备（如网卡）建立回调关系，事件发生时，会将事件放到rdlist中

op
- EPOLL_CTL_ADD：注册新的socket到epollfd
- EPOLL_CTL_MOD：修改已经注册的socket的监听事件
- EPOLL_CTL_DEL：从epollfd中删除一个socket

- epoll_wait
传入epollfd，代表这次系统调用监听的socketfd集合是eventpoll中指定的socket

函数原型：
```
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
```

	- epfd，epoll create创建的eventpoll的描述符
	- events，期望从内核得到的事件集合
	- max events，告知内核期望得到的返回events的大小
	- timeout，超时时间

开始阻塞等待描述符的数据
与select,epoll区别
- epollfd不会由用户态拷贝到内核态，用户态和内核态共享这块内存
- 一旦epollfd注册的某一个socket描述符有数据了，由于在epollctl中已经建立了回调关系，一旦有数据设备就会回调，将事件加入到rdlist中
- 当到了epollwai的超时时间，检查rdlist，若不为空则将其拷贝到入参中的events数组即可，这里使用了共享内存
- epoll_wait函数会返回，带有返回值，返回值即为有数据的描述符个数n，因此只需遍历前n个events即可

两种触发方式：
- 水平触发
只要这个文件描述符还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作
- 边缘触发
通知之后进程必须立即处理事件。下次再调用 epoll_wait() 时不会再得到事件到达的通知。这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符。