---
title: 【MQ】可靠消息
date: 2017-12-08 21:55:53
tags: MQ
categories: MQ
---

初始【MQ】最后说到默认情况下，消息发送后 MQ 不会向发送方确认消息到达，也不会进行持久化处理。即在发送方眼里消息只要发出去，就不再关心消息消息了。这确实做到了生产者与 MQ 的解耦，并且效率很高。但缺点也非常明显，无法确定消息投递是可靠的：

- 正在运行的 MQ 宕机后，无法恢复已发送的消息（持久化问题）
- 没有匹配的 queue，那么消息将被 exchange 直接丢弃，而发送方对此毫不知情（确认问题）
- 消息发送过程中在网络中丢失，发送方毫不知情（确认问题）

Rabbit MQ 是被设计为金融行业服务的，在这些方面当然有考虑。本文将从持久化和消息确认两方面来了解 Rabbit MQ 的可靠消息实践。

## 持久化

为了确保消息在 MQ 各个环节的不丢失，需要将 exchange, queue, 投递方式都进行持久化声明。具体持久化的方式很简单，调用 API 就可以了。

### exchange 持久化

exchange 声明时，将 durable 设置为 true 就可以了。这顺便看一下 exchange 创建方法

```java
Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable) 
  throws IOException;

Exchange.DeclareOk exchangeDeclare(String exchange, String type, boolean durable, 
                                   boolean autoDelete,Map<String, Object> arguments) 
  throws IOException;

Exchange.DeclareOk exchangeDeclare(String exchange, String type) 
  throws IOException;

Exchange.DeclareOk exchangeDeclare(String exchange,  // 交换器名称
                                   String type,  // 交换器类型
                                   boolean durable, // 是否持久化
                                   boolean autoDelete,  // 是否自动删除
                                   boolean internal,  // 内部
                                   Map<String, Object> arguments  // 其他构造参数
                                  ) throws IOException;

// 等价于 exchangeDeclare 方法设置 nowait 参数
void exchangeDeclareNoWait(String exchange, String type, boolean durable, boolean autoDelete,
                           boolean internal, Map<String, Object> arguments) 
  throws IOException;

// 被动声明队列，声明前先检查
Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException;

```

exchange 声明持久化后只能确保重启后 exchange 重新创建。否则 exchange 将丢失，生产者就无法正常发送消息了。

### queue 持久化

queue 持久化也是一样的套路，将 durable 设置为 true 就可以了。queue 创建的 AIP：

```java
Queue.DeclareOk queueDeclare() throws IOException;

Queue.DeclareOk queueDeclare(String queue,  // queue 名称 
                             boolean durable,  // 持久化
                             boolean exclusive,  // 排他队列
                             boolean autoDelete,  // 自动删除
                             Map<String, Object> arguments  // 其他构造参数
                            ) throws IOException;

void queueDeclareNoWait(String queue, boolean durable, boolean exclusive, boolean autoDelete, 
                        Map<String, Object> arguments) throws IOException;

Queue.DeclareOk queueDeclarePassive(String queue) throws IOException;
```

对 durable 没什么好说的，确保重启后 queue 重新创建，但消息无法恢复，消息的持久化依赖于投递方式的持久化。

注意一下 exclusive 参数：一个队列被声明为排他队列，该队列仅对首次申明它的连接可见，并在连接断开时自动删除：

1. 排他队列是基于连接可见的，同一连接的不同信道是可以同时访问同一连接创建的排他队列；
2. “首次”，如果一个连接已经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同；
3. 即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除的，这种队列适用于一个客户端发送读取消息的应用场景。

### 投递方式持久化声明

套路基本一致，还是看 API：

```java
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) 
  throws IOException;

void basicPublish(String exchange, String routingKey, boolean mandatory, BasicProperties props,
                  byte[] body)throws IOException;

void basicPublish(String exchange,  // 交换器
                  String routingKey,  // routing key
                  boolean mandatory,  // 消息确认
                  boolean immediate,  // 废弃
                  BasicProperties props,  // 参数
                  byte[] body  // 消息有效负载
                 ) throws IOException;
```

持久化的参数包含在 BasicProperties 定义中：

```java
public static class BasicProperties extends AMQBasicProperties {
    private String contentType;  // 消息类型
    private String contentEncoding;  // 编码
    private Map<String, Object> headers;
    private Integer deliveryMode;  // 持久化。1：非持久化；2：持久化
    private Integer priority;  // 优先级
    private String correlationId;
    private String replyTo;  // 反馈队列
    private String expiration;  // expiration到期时间
    private String messageId;
    private Date timestamp;
    private String type;
    private String userId;
    private String appId;
    private String clusterId;
    // 省略方法   
}    
```

BasicProperties 的构造除了提供默认的方法外，对常用的参数可以直接获得，还支持使用 builder 模式构造。

**如果单独持久化投递方式，重启后因为交换器、队列已不存在所以毫无意义**

### 持久化的影响

- 性能

  《Rabbit MQ 实战》 一书在说明持久化对性能影响时，举例：“使用持久化机制而导致消息吞吐量降低至少 10 倍的情况并不少见”。这个说法还是很让我震惊的，很好奇 Rabbit MQ 的持久化策略是怎么做的影响这么大，还是说非持久化策略太优秀了，以至于磁盘性能极大影响了整体吞吐量。这里挖个坑，争取以后看看内部实现吧，毕竟 erlang 对我是个大问题。

- 集群模式下工作的不好

  暂时不清楚集群模式下的影响，先 mark 一下

- 依旧无法 100% 数据不丢失

  即使 exchange，queue，投递方式都进行持久化声明依旧不能做到 100% 数据不丢失，原因有二：

  1. Rabbit MQ 不是为每条消息进行 fsync（同步 IO） 处理

     依旧可能出现挂掉时有消息没有持久化的情况，解决有两种方式：镜像队列和消息确认

  2. 看到网上有提到 erlang 写文件的实时问题，不懂，先 mark，待求证

## 消息确认

消息确认可以分为生产者确认消息正确投递和消费者确认消息正确接收，对  Rabbit MQ 有三种更具体的情况：

- confire/事务：确认消息到达 broker，避免消息在生产者发出后丢失
- 客户端 ACK：确认消费者接收消息，避免消息在消息队列发出后丢失
- mandatory/immediate：确认消息到达队列，避免到达交换器后找不到队列而丢弃

### 事务/confire

#### 事务

确认消息成功被 exchange 接收。事务是 AMQP 协议内定义的， Rabbit MQ 也做了相应的实现。与事务相关有三个方法，具体使用的模板：

```java
try {
  channel.txSelect();
  channel.basicPublish(...);
  channel.txCommit();
} catch (Exception e) {
  e.printStackTrace();
  channel.txRollback();
}
```

事务缺点：最大的问题是执行前后需要开启事务，提交/回滚事务，而这几个过程又必须是同步的因此会造成很大的性能问题

#### confire

confire 是 Rabbit MQ 为解决事务性能问题设计的确认机制，主要的做法是为每条消息都设置唯一 ID 且 ID 以 1  为步长生序，MQ 通过发送 ACK, NACK 异步确认消息是否到达交换器。

网上普遍对 confire 的描述都集中在异步性上。除了异步，可以设置 basic.ack 的 multiple 域进行累计确认，这有点 TCP 的确认方式。

confire 最大的问题是无法回滚，导致生产者本身也不确定消息是否放成功。如果程序需要实现类似回滚功能，则维护一个 unconfire 消息的集合，每次收到 ACK/NACK 时更新集合（还需要考虑是否是累计确认）

我使用了三种方式实现 confire 并进行对比：

- 对每条消息要求接收对应的 confire 消息
- 对一组消息要求接收一条 confire 消息
- 使用监听器完全异步的接收 confire 消息

不出意外的第三种方式的性能是最好的。

### 客户端 ACK

声明队列时指定 noAck 参数：

- noAck=false：Rabbit MQ 向消费者发出消息后等待消费者显式发出 ack 信号后才移除消息
- noAck=true：Rabbit MQ 向消费者发出消息后立即移除消息

当设置队列 noAck 为 false 时，客户端必须根据消息的处理情况向 MQ 反馈，默认情况下 会自动确认。如果希望手动确认需要关闭自动确认。

客户端除了 ACK 为还可以向 MQ 反馈其他信息，反馈的 API 分别有：

- channel.basicAck：向 MQ 确认消息正确接收
- channel.basicRecover：向 MQ 确认消息需要重发，可以根据参数重发给当前消费者或重新入队
- channel.basicReject：向 MQ 确认消息退回
- channel.basicNack：向 MQ 确认批量退回消息，可以根据参数选择是否批量

### mandatory/immediate

#### mandatory

mandatory 设置为 true 时：MQ 至少将该消息路由到至少一个队列中，否则将消息返还给生产者

mandatory 实现时只需要：

1. 投递消息时设置 mandatory 参数为true

   ```java
   void basicPublish(String exchange,  // 交换器
                 String routingKey,  // routing key
                 boolean mandatory,  // 消息确认
                 boolean immediate,  // 废弃
                 BasicProperties props,  // 参数
                 byte[] body  // 消息有效负载
                ) throws IOException;
   ```

2. 设置监听器

   ```java
   channel.addReturnListener(new ReturnListener() {
       public void handleReturn(int replyCode, String replyText, String exchange,
                                String routingKey, AMQP.BasicProperties basicProperties,
                                byte[] body) throws IOException {
                                  // TODO
                                }
   });
   ```

当消息没有被正确路由到至少一个队列时，AMQP协议会返回对应消息，监听器内的代码将被调用；

**当消息正确投递，什么也不发生**

#### immediate

**Rabbit MQ 3.0 之后已移除**。设置为 true 时：消息路由到 queue 前，如果 queue 有消费者，则马上将消息投递给 queue，否则直接把消息返还给生产者，消息不再入队。

---

参考：

《Rabbit MQ 实战》

[RabbitMQ(二)：mandatory标志的作用](http://www.cnblogs.com/520playboy/p/6925196.html)

[RabbitMQ：Publisher的消息确认机制](https://github.com/pzxwhc/MineKnowContainer/issues/49)

[RabbitMQ之mandatory和immediate](http://blog.csdn.net/u013256816/article/details/54914525)

 [RabbitMQ之消息确认机制（事务+Confirm）](http://blog.csdn.net/u013256816/article/details/55515234)

[rabbitMq生产者角度:消息持久化、事务机制、PublisherConfirm、mandatory](http://blog.csdn.net/u014045580/article/details/70311746)
