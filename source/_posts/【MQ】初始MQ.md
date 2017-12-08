---
title: 【MQ】初始MQ
date: 2017-12-08 21:48:26
tags: MQ
categories: MQ
---

接触 MQ 之前简单的理解消息队列就是一个理论上无限大的线性表，接触后发现 MQ 支持的功能远不止这些。MQ 的功能可以概括为：削峰填谷，异步解耦。

从模型上看，MQ 模型不是狭义上的 C/S 模型，而是消息服务投递模型：

- 在程序角度：当程序连接到 RabbitMQ 时必须决定自己是发送者还是接受者
- 在 MQ 角度：MQ 及接收消息，又发送消息

AMQP（高级消息队列协议）是对 MQ 最抽象的描述。

## AMQP

AMQP 定义了一个 MQ 的几个组件，官方的描述还是比较晦涩的，我以自己的理解描述所以可能不够准确：

- Server（broker）：MQ 服务器
- Exchange：一个功能强大 router，不做消息的存储，单纯转发给 MQ
- Message Queue：消息队列，具体存储未被消费的消息
- Message：消息
- Binding：关联 Exchange  和 Message Queeu 的路由表
- Connection：链接，TCP 链接
- Channel：子链接，复用 Connection
- Command：命令
- Virtual Host：服务器创建的 mini 版的 MQ

### Exchange & Binding

这两个东东算是 MQ 核心功能的实现组件，网上描述的我觉得不是很清楚。

可以把 exchange 当路由器理解，把 binding 当路由表理解。路由器根据路由表把数据从路由器路由到一下节点，exchange 根据 binding 把消息从 exchange 路由到 queue。exchange 的核心功能是路由转发，而路由转发的依据是 binding。把 binding 当路由表的话，那么这个路由表有三项：

- exchange name
- queue name
- router key

三者的关系需要在实际生产、消费消息之前完成绑定。而后消息到达 exchange 后根据 routing key 路由到指定 queue。而 exchange  有多种不同实现，不同实现的 exchange 根据 routing key 的路由方式不同，适用于不同场景。

## 典型场景

以下例子从 Rabbit MQ 官网给搬运，[传送门](http://www.rabbitmq.com/tutorials/tutorial-one-java.html)。

### direct

direct 类型的交换器严格根据消息头的 exchange name, queue name, router key 将消息路由到对应的队列

#### 消息投递到一个队列

![direct1](http://www.rabbitmq.com/img/tutorials/python-one.png)

所有消息由默认交换器根据消息头 queue name 投递到队列。没有声明交换器，自动将队列绑定到了默认交换器。下面代码的第二个参数很容易被当做 queue name，实际上这个字段是 routing key，发送方是不关心 queue 的。

```java
channel.queueDeclare(QUEUE_NAME, false, false, false, null);  // 声明队列
channel.basicPublish("",  // exchange name，空则投递到默认交换器
                     QUEUE_NAME,  // 以 queue_name 作为 routing key
                     null, 
                     message.getBytes());
```

#### 消息投递到一个队列由多个消费者消费

![direct2](http://www.rabbitmq.com/img/tutorials/python-two.png)

可以用于负载均衡的生产者消费者模型，每个消息正常只被消费一次。

投递过程与上一个一样，队列的消息同时由多个消费者消费

#### 消息有选择的分散到多个队列

![direct3](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)

```java
channel.exchangeDeclare(EXCHANGE_NAME, "direct");  // 声明 direct 类型交换器
String queueName = channel.queueDeclare().getQueue();  // 声明随机队列，并获取该队列名字
channel.queueBind(queueName, EXCHANGE_NAME, ROUTING_KEY);  // 绑定

// 发送
channel.basicPublish(EXCHANGE_NAME, 
                     ROUTING_KEY, 
                     null, 
                     message.getBytes());
```

完整的交换器，队列声明并绑定，消息根据绑定信息投递到对应队列。

一个交换器与多个队列使用相同的 routing key 进行绑定，当该 routing key 消息发送至交换器可以形成广播的形式。

### fanout

![fanout](http://www.rabbitmq.com/img/tutorials/exchanges.png)

```java
channel.exchangeDeclare(EXCHANGE_NAME, "fanout");  // 声明 fanout 类型交换器
String queueName = channel.queueDeclare().getQueue();  // 创建非持久的，唯一的，自动删除的队列
channel.queueBind(queueName, EXCHANGE_NAME, "");  // 绑定队列与交换器，不要 routing key

// 发送
channel.basicPublish(EXCHANGE_NAME, 
                     "",  // routing key
                     null, 
                     message.getBytes());
```

交换器收到的消息广播至所有绑定的队列，绑定不需要给定 routing key

### topic

![topic](http://www.rabbitmq.com/img/tutorials/python-five.png)

```java
channel.exchangeDeclare(EXCHANGE_NAME, "topic");  // 声明 topic 类型交换器
String queueName = channel.queueDeclare().getQueue();  // 创建非持久的、唯一的、自动删除的队列
channel.queueBind(queueName, EXCHANGE_NAME, ROUTING_KEY);  // 绑定队列，交换器，路由键

// 发送
channel.basicPublish(EXCHANGE_NAME, 
                     ROUTING_KEY, 
                     null, 
                     msg.getBytes());
```

编码过程与使用 direct 交换器的完整过程一直，但是 routing key 可以使用通配符：

- `*` 将 . 视为分隔符进行匹配
- `#`将任意字符串视为关键字匹配

## 其他

- 消息发后即忘：消息单向传递，默认并不会向发送方确认发送，也不会做持久化
- prefetch count：在 direct 的第二场景下，消息会被平均分配给各个消费者，而不考虑消费者的消费能力。可以使用设置 Prefetch count 保持各消费者负载均衡
- binding key：有的文章将 bindling 中使用的 routing key 也称作 binding key，我统一称为 binding key 了
- 相关 demo [传送门](https://github.com/zhanghTK/rabbit-mq)
