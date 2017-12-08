---
title: Docker结构
date: 2017-12-08 22:07:19
tagsi: 容器
categories: 容器
---

## 容器介绍

### 容器与虚拟机

- 容器：应用程序；依赖
- 虚拟机：应用程序；依赖；操作系统

![docker_vm.jpg](https://ooo.0o0.ooo/2017/06/21/594a1480eb090.jpg)

### 容器解决的问题

简化打包、部署，使应用具备了超强的可移植能力

对于现在多服务的应用往往依赖多个组件（例如：MQ,DB,Cache等），整个开发周期又需要部署多个环境（开发，测试，正式等），这就为运维带来极大的不便：

![c3786681cc644e6d8a6204cdffc49aa3_th.jpeg](https://ooo.0o0.ooo/2017/06/21/594a17934f0dd.jpeg)

上面的图有两个变量：

1. 应用组件
2. 服务器环境

容器能做的就是就是为应用组件提供一个基于容器的标准化环境，让容器可以运行在几乎所有操作系统上

![54e8e6a8d2ec4b2d8acba239d323dfb9_th.jpeg](https://ooo.0o0.ooo/2017/06/21/594a19569b110.jpeg)

好处：

- 隔离：容器环境与宿主环境隔离
- 重用：同一个组件只需要创建一次运行环境就可以在其他机器上运行
- 一致：只需要配置好标准的 runtime 环境，服务器就可以运行任何容器

## Docker组成

核心组件：

1. Docker 客户端 - Client
2. Docker 服务器 - Docker daemon
3. Docker 镜像 - Image
4. Registry
5. Docker 容器 - Container

![docker_architecture.jpg](https://ooo.0o0.ooo/2017/06/21/594a29f4c365c.jpg)

## Docker启动过程

上篇文章最后提到了Docker的安装并运行了httpd：`sudo docker run -d -p 80:80 httpd`

结合Docker的组成，说明一下在容器启动过程这些组件是怎么协作的：

1. docker client调用docker daemon请求启动一个容器
2. docker daemon会向host请求创建容器
3. host创建一个空的容器
4. docker daemon检查本机docker镜像文件（如果没有，则到Registry下载）
5. 将镜像文件加载到容器中
