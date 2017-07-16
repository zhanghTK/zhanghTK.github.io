---
title: 【Netty】IO模型
date: 2017-07-17 00:14:18
tags: 
  - Netty
  - Java
categories: Netty
---
## 网络 IO 模型

UNIX 网络编程对 IO 模型分类， UNIX 提供 5 种 IO 模型

在这 5 个 IO 模型之前，先看看这五个 IO 模型最后的关注点：

### 用户 & 内核空间

内核：操作系统的核心，独立于普通的应用程序，可以访问受保护的内存空间，有访问底层硬件设备的所有权限

为了保证用户进程不能直接操作内核（kernel），保证内核的安全，操作系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。

### 缓存 IO

数据会先拷贝到操作系统内核的缓冲区，然后才从操作系统内核的缓冲区拷贝到应用程序的地址空间

缺点：数据需要在应用程序空间和内核空间进行数据拷贝，影响效率

有了上面几个基本概念下面看看网络 IO

## 网络 IO

IO 操作的本质是对数据的读写。因为 IO 缓存的缘故，一个 IO 操作需要分为两步，以读操作为例：

1. 等待数据准备
2. 将数据从内核空间拷贝到用户空间

对网络 IO 而言，上面的两步可以更具象的表述为：

1. 等待网络数据分组到达，然后将其复制到内核缓冲区
2. 把数据从内核缓冲区复制到应用空间缓冲区

由于网络传输的不确定性，第一步操作可能成为最耗时的过程（可能比后面的业务逻辑计算都更为耗时）。

下面看 5 个 IO 模型是如何解决问题的，相关的图就不放了网上一大把：

### 阻塞 IO 模型

这个模型下，其实就是不解决任何问题，第一步，第二部都是阻塞的。

以 socket 操作为例，进程空间调用 recvfrom，其系统调用会直到数据包到达并且复制到应用进程缓冲区才返回。

### 非阻塞 IO 模型

使用轮询操作，避免第一步操作完全被阻塞。

进程调用 recvfrom，如果缓冲区没有数据立即返回一个 EWOULDBLOCK，通过轮询操作检查内核缓冲区是否有数据到来。

### IO 复用模型

Linux 提供系统调用，进程将文件描述符传递给系统调用，若第一步未就绪则阻塞在系统调用上。

Linux对 IO 复用模型提供的系统调用：

- select：顺序描述所有文件描述符，文件描述符最多持有 1024 个
- poll：类似于select，只是去除了文件描述符数组长度的限制
- epoll：利用mmap技术避免了这些复制和遍历操作

**Java NIO多路复用器 Selector 就是基于 epoll 的多路复用技术实现的**

### 信号驱动 IO 模型

应用进程建立 SIGIO 信号处理程序，调用 sigaction 后返回。IO 第一步完成后通知程序。

### 异步 IO

告知内核启动某个操作，并让内核在整个操作完成后通知我们。

### IO 模型对比

![1486392482597_6.png](https://i.loli.net/2017/07/16/596b64fa04869.png)

## Java 对于 IO 模型的实现

### JDK 支持

- JDK 1.0 ~ JDK 1.3：传统的 BIO，基于阻塞 IO 模型
- JDK 1.4 ~ JDK 1.5：加入 NIO，基于 IO 复用模型
  - JDK 1.4 ~ JDK 1.5 update10：Selector 基于 select/poll 模型实现
  - JDK 1.5 update & Linux core 2.6 以上：Selector 使用 epoll 模型实现
- JDK 1.7 ~ JDK 1.8（目前）：加入 NIO 2.0（AIO），增加异步套接字通道

### 使用

- [阻塞 IO](https://github.com/zhanghTK/netty/tree/master/src/main/java/tk/zhangh/netty/ch1/bio)
- [伪异步 IO](https://github.com/zhanghTK/netty/tree/master/src/main/java/tk/zhangh/netty/ch1/paio)
- [非阻塞 IO](https://github.com/zhanghTK/netty/tree/master/src/main/java/tk/zhangh/netty/ch1/nio)
- [异步 IO](https://github.com/zhanghTK/netty/tree/master/src/main/java/tk/zhangh/netty/ch1/aio)

---

参考：

1. 《Netty 权威指南》（第2版）：第1章，第2章
2. [聊聊并发，Part 1：IO模型](http://blog.leanote.com/post/joesay/Concurrency-Model-Part-1-IO-Concurrency)
3. [聊聊Linux 五种IO模型](http://www.jianshu.com/p/486b0965c296)
