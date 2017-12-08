---
title: 【MQ】镜像队列
date: 2017-12-08 21:57:51
tags: MQ
categories: MQ
---

前面提到持久化和消息确认可以确保消息的可靠，但在默认情况下 MQ 的可靠性完全没有保障，通过集群的方式确保服务的可靠往往是高可用的第一步。本文记录一下 Rabbit MQ 的集群和镜像。

## 集群

集群能够带来的好处主要有两点：

1. 允许消费者和生产者在 Rabbit MQ 节点崩溃的情况下继续运行
2. 通过添加更多节点线性的扩展消息通信吞吐量

### 模式

在介绍集群之前先看看从单节点到集群的模式异同：

- 相同：任何模式下节点内部都需要维护基本元数据信息：队列元数据、交换器元数据、绑定元数据、vhost 元数据。不同模式

- 差异：

  - 单一节点模式：

    默认基本元数据信息存储在内存，被标记持久化的队列和交换器已经它们的绑定存储到磁盘

  - 普通集群模式

    ![cluster.PNG](https://i.loli.net/2017/08/23/599ce92322b59.png)

    ![cluster.PNG](https://i.loli.net/2017/08/23/599ce7656d746.png)

    除了基本元数据，还有集群相关元数据。与单一节点模式的不同主要在集群对队列，交换器，数据存储的差异。

    队列：只会在单个节点而不是所有节点上创建完整队列信息，其余节点只保存队列的元数据。虽然只在一个节点保存完整队列，但消息可以在不同节点之间临时传输（消费者感知到每个节点都有完整的队列）。如果保存队列的单个节点挂了，则消费者对其订阅丢失，即将投递到该队列的信息消息也丢失。如果挂掉的队列是持久化队列则无法重新创建队列，必须恢复该队列

    交换器：交换器实质是一张查询表（消息的转发路由是由信道完成），集群内所有的节点拥有所有交换器的信息

    数据存储：分为内存节点和硬盘节点，硬盘节点防止重启后元数据信息丢失。元数据的创建更新在集群所有节点操作完成后才返回。集群下要求任何时刻集群中至少有一个磁盘节点，如果唯一的磁盘节点挂了，集群只能路由消息但不能创建更新元数据

  - 镜像队列

    ![image_queues.PNG](https://i.loli.net/2017/08/23/599ce8d6ae6a3.png)

    镜像不在单独存在在唯一节点，而是冗余在多个节点

## 镜像队列

因为普通集群模式相对基础，而镜像队列复杂，这里重点讨论一下镜像队列。

### 概述

队列镜像通常包括一个 master 节点和多个 slave 节点，每个节点都复制队列数据。当一个节点失效时，可以自动切换到另一个节点确保可用。在镜像队列模式下，除了 publish 外的所有动作都只会向 master 发送，然后 master 将命令执行的结果广播为所有 slave，publish 到镜像的所有消息总是被直接 publish 到所有 slave 之上（类似与 fanout 交换器）。

### 原理

#### 普通队列结构

普通队列由两部分组成：

- AMQQueue：主要负责 AMQP 协议的逻辑功能
- BackingQueue：存储消息

对于 BackingQueue 又由五个子队列组成：Q1, Q1, Delta, Q3, Q4，MQ 的消息进入队列后会随着系统负载在队列中流动，BackingQueue 中的消息可以分为四个状态：

- Alpha：消息的内容和索引都在内存中，Q1 和 Q4 的状态
- Beta：消息的内容在硬盘，消息的索引在内存，Q2 和 Q3 的状态
- Gamma：消息内容在硬盘，消息的索引在硬盘和内存都有，Q2 和 Q3 的状态
- Delta：消息的内容和消息的索引都在硬盘上，Delta 状态

对于持久化的消息，消息内容和消息索引都必须先保存到磁盘上，才会处于上述状态中的一种，而Gamma状态的消息只有持久化的消息才会有该状态。

从 Q1 到 Q4，基本的经历是由内存到硬盘再到内存的设计，分层的好处使得整个队列有很好的弹性:

- 当队列负载很高的情况下，能够通过将一部分消息由磁盘保存来节省内存空间
- 当负载降低的时候，这部分消息又渐渐回到内存，被消费者获取

引起消息流动的两种情况：消费者获取消息，内存不足

当系统处于正常负载，对消息的消费速度不小于接收速度，对于非消息极可能只会有 Alpha 状态。对于持久化消息一定会进入 gamma 状态。如果开启 confirm 机制，只有到了这个阶段才会确认消息已经被接受，当消费足够快且内存充足消息不会继续走到下一状态。

当系统处于高负载，已接受的消息不能很快被消费，这些消息就会进入很深的队列中去，增加处理每个消息的平均开销。因为平均开销增加，处理速度更慢，由此恶性循环，使得系统的处理能力大大降低。

改善措施：

1. 进行流程控制
2. 增加 prefetch 的值，一次发送更多消息给消费者
3. 采用 multiple ack

#### 镜像队列结构

在镜像队列中 AMQQueue 仍旧负责 AMQP 协议的逻辑功能，而 backing_queue 已不是简单的单节点 backing_queue 了。

backing_queue 是由 master 和 slave 节点组成的特殊 backing_queue，所有对 mirror_queue_master 的操作，会通过 GM 同步到 slave 节点，slave 节点上 mirror_queue_slave 负责回调，master 节点上 coordinato 负责回调。

镜像队列对消息的操作：

- basic.publish 操作：操作直接同步到所有节点
- 其他操作：通过 master 操作，由 master 将结果给 slave

##### GM

GM(Guarenteed Multicast)，实现可靠组播通讯协议的模块，确保组播消息的原子性：

- 将所有节点形成一个收尾相连的循环链表
- 当有节点新增时，相邻的节点保证当前广播的消息会复制到新的节点上
- 当有节点失效时，相邻的节点会接管保证本次广播的消息会复制到所有节点
- 消息从master节点对应的gm发出后，顺着链表依次传送到所有节点

### 镜像队列细节备忘

镜像队列细节太多，这里整理网上一个注意事项：

1. 镜像队列不能作为负载均衡使用，因为每个操作在所有节点都要做一遍

2. ha-mode 参数与 durable, declare 对 exclusive 队列都不生效。exclusive队列是连接独占的，当连接断开，队列自动删除，这两个参数对exclusive队列没有意义

3. 将新节点加入已存在的镜像队列时，默认情况下 ha-sync-mode=manual，镜像队列中的消息不会主动同步到新节点，除非显式调用同步命令。当调用同步命令后，队列开始阻塞，无法对其进行操作，直到同步完毕。当 ha-sync-mode=automatic 时，新加入节点时会默认同步已知的镜像队列。由于同步过程的限制，所以不建议在生产环境的active队列(有生产消费消息)中操作

4. 每当一个节点加入或者重新加入(例如从网络分区中恢复回来)镜像队列，之前保存的队列内容会被清空

5. 镜像队列有主从之分，一个主节点(master)，0个或多个从节点(slave)。当 master 宕掉后，会在 slave中 选举新的master。选举算法为最早启动的节点

6. 当所有slave都处在(与master)未同步状态时，并且 ha-promote-on-shutdown policy 设置为 when-syned(默认) 时，如果 master 因为主动的原因停掉，比如是通过 rabbitmqctl stop 命令停止或者优雅关闭 OS，那么slave不会接管 master，也就是说此时镜像队列不可用

   但是如果master因为被动原因停掉，比如 VM 或者 OS crash了，那么 slave 会接管 master。这个配置项隐含的价值取向是优先保证消息可靠不丢失，放弃可用性。

   如果 ha-promote-on-shutdown policy 设置为 alway，那么不论 master 因为何种原因停止，slave 都会接管 master，优先保证可用性

7. 镜像队列中最后一个停止的节点会是 master，启动顺序必须是 master 先起，如果 slave 先起，它会有 30 秒的等待时间，等待 master 启动，然后加入 cluster。

   当所有节点因故(断电等)同时离线时，每个节点都认为自己不是最后一个停止的节点。要恢复镜像队列，可以尝试在 30 秒之内同时启动所有节点

8. 对于镜像队列，客户端basic.publish操作会同步到所有节点；而其他操作则是通过master中转，再由master将操作作用于salve。比如一个basic.get操作，假如客户端与slave建立了TCP连接，首先是slave将basic.get请求发送至master，由master备好数据，返回至slave，投递给消费者

9. 当 slave 宕掉时，除了与 slave 相连的客户端连接全部断开之外，没有其他影响。

   当 master 宕掉时，会有以下连锁反应：

   1. 与 master 相连的客户端连接全部断开。
   2. 选举最老的 slave 为 master。若此时所有 slave 处于未同步状态，则未同步部分消息丢失。
   3. 新的 master 节点 requeue 所有 unack 消息，因为这个新节点无法区分这些 unack 消息是否已经到达客户端，亦或是 ack 消息丢失在到老master的通路上，亦或是丢在老 master 组播 ack 消息到所有 slave 的通路上。所以处于消息可靠性的考虑，requeue 所有 unack 的消息。此时客户端可能受到重复消息。
   4. 如果客户端连着 slave，并且 basic.consume 消息时指定了x-cancel-on-ha-failover参数，那么客户端会收到一个 Consumer Cancellation Notification 通知，Java SDK中会回调 Consumer 接口的handleCancel() 方法，故需覆盖此方法。如果不指定 x-cancel-on-ha-failover 参数，那么消费者就无法感知 master 宕机，会一直等待下去

### 镜像队列的恢复

前提：两个节点 A 和 B 组成以镜像队列

- 场景一：A 先停，B 后停

  该场景下 B 是 master，只要先启动 B，再启动 A 即可。或者先启动 A，再在 30s 之内启动 B 即可恢复镜像队列。如果没有在 30s 内恢复 B，那么 A 自己就停掉自己

- 场景二：A，B 同时停

  该场景可能是由掉电等原因造成，只需在 30s 之内连续启动 A 和 B 即可恢复镜像队列

- 场景三：A 先停，B 后停，且 A 无法恢复

  因为 B 是 master，所以等 B 起来后，在 B 节点上调用 rabbitmqctl forget_cluster_node A 以解除 A 的 cluster 关系，再将新的 slave 节点加入 B 即可重新恢复镜像队列

- 场景四：A 先停，B 后停，且 B 无法恢复

  此时 B 是 master，所以直接启动 A 是不行的，当 A 无法启动时，也就没办法在 A 节点上调用 rabbitmqctl forget_cluster_node B。新版本中，forget_cluster_node 支持 --offline 参数，offline 参数允许 rabbitmqctl 在离线节点上执行 forget_cluster_node 命令，迫使 RabbitMQ 在未启动的 slave 节点中选择一个作为 master。当在 A 节点执行 rabbitmqctl forget_cluster_node --offline B 时，RabbitMQ 会 mock 一个节点代表 A，执行 forget_cluster_node 命令将 B 剔出 cluster，然后 A 就能正常启动了。最后将新的 slave 节点加入 A 即可重新恢复镜像队列

- 场景五：A 先停，B 后停，且 A 和 B 均无法恢复，但是能得到 A 或 B 的磁盘文件

  这个场景更加难以处理。将A或B的数据库文件（$RabbitMQ_HOME/var/lib目录中）copy至新节点C的目录下，再将 C 的 hostname 改成 A 或者 B 的 hostname。如果 copy 过来的是 A 节点磁盘文件，按场景四处理，如果拷贝过来的是 B 节点的磁盘文件，按场景三处理。最后将新的 slave 节点加入 C 即可重新恢复镜像队列

- 场景六：A 先停，B 后停，且 A 和 B 均无法恢复，且无法得到 A 和 B 的磁盘文件

  跑路吧

---

参考：

[RabbitMQ分布式集群架构和高可用性（HA）](http://chyufly.github.io/blog/2016/04/10/rabbitmq-cluster/)

[rabbitmq——镜像队列](https://my.oschina.net/hncscwc/blog/186350)

[RabbitMQ源码分析 - 队列机制](http://jzhihui.iteye.com/blog/1582294)

[RabbitMQ系列三 （深入消息队列）](http://backend.blog.163.com/blog/static/202294126201322511327882/)

[RabbitMQ镜像队列的故障恢复](http://fengchj.com/?p=2273)


