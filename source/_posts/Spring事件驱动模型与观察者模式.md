---
title: Spring事件驱动模型与观察者模式
date: 2017-08-14 23:25:10
tags:
 - Java
 - Spring
categories: Spring
---
## 事件驱动模式

关于事件驱动模式，这又是个大话题了，不是本文的核心关注点，开涛的博客中以用户注册为例子做了[介绍](http://jinnianshilongnian.iteye.com/blog/1902886)，这里就不做搬运工了。总结起来，事件驱动的关键点：松散耦合对象间的一对多依赖关系。

具体到实现可以在语言层面使用观察者模式进行对象解耦，也可以使用消息队列对服务进行解耦，下面看看观察者模式。

## 观察者模式

### JDK 支持

JDK 中直接提供了对观察者模式的抽象：

- 被观察者：java.util.Observable
- 观察者：java.util.Observer

JDK 观察者模式的简单使用 [demo](https://github.com/zhanghTK/JPattern/tree/master/src/main/java/tk/zhangh/pattern/behavior/observer)

值得注意的是：JDK 的观察者模式同时支持了 push 和 pull 两种模式，具体来说：

push：每当被观察者（主题）更新，被观察者主动推送消息到各个观察者

pull：每当被观察者（主题）更新，被观察者仅通知观察者已更新，由观察者自行决定更新策略

### push & pull

push 的优点，被观察者（主题）可控：

1. 可以实时推送，事件发生后第一时间即可触发通知操作
2. 延迟推送，避开繁忙时间、明确事件发生顺序

pull 的优点：

1. 减轻被观察者（主题类）负担
2. 更新策略策略自定义，何时更新，更新哪些内容都由观察者自定义
3. 权限管控，被观察者（主题类）可以根据不同主题集中处理权限管控

网上看到关系新浪微博的推/拉架构的讨论：

[软件架构模式-事件驱动不错哦](http://www.jianshu.com/p/dfb7fd88cd33)

[微博feed系统的推(push)模式和拉(pull)模式和时间分区拉模式架构探讨](http://www.cnblogs.com/sunli/archive/2010/08/24/twitter_feeds_push_pull.html)

## Spring 事件机制

其实之前在[【SpringBoot】监听器篇](http://zhangh.tk/2017/07/05/%E3%80%90SpringBoot%E3%80%91%E7%9B%91%E5%90%AC%E5%99%A8%E7%AF%87/)，[【Spring】容器刷新](http://zhangh.tk/2017/07/16/%E3%80%90Spring%E3%80%91%E5%AE%B9%E5%99%A8%E5%88%B7%E6%96%B0/) 中已经把 Spring 中的使用到的场景说的七七八八了，这里做一下补充和归纳，合起来阅读效果更佳。

Spring 事件体系包括三个组件：事件，事件监听器，事件广播器。

### 事件

![Spring 事件](https://i.loli.net/2017/08/14/5991c0618bfb4.jpeg)

Spring 默认对 ApplicationEvent 事件提供了如下实现：

- ContextStoppedEvent：ApplicationContext停止后触发的事件；
- ContextRefreshedEvent：ApplicationContext初始化或刷新完成后触发的事件；
- ContextClosedEvent：ApplicationContext关闭后触发的事件；（如web容器关闭时自动会触发spring容器的关闭，如果是普通java应用，需要调用ctx.registerShutdownHook();注册虚拟机关闭时的钩子才行）

- ContextStartedEvent：ApplicationContext启动后触发的事件；

Spring Boot 额外支持的事件类型：

- ApplicationStartingEvent
- ApplicationEnvironmentPreparedEvent
- ApplicationFailedEvent
- ApplicationPreparedEvent

### 事件监听器

![Spring事件监听器](https://i.loli.net/2017/08/14/5991bfaf3c0a7.jpeg)

Spring 事件监听器注册：[Spring 事件监听器注册](http://zhangh.tk/2017/07/16/%E3%80%90Spring%E3%80%91%E5%AE%B9%E5%99%A8%E5%88%B7%E6%96%B0/#registerListeners)

Spring Boot 事件监听器：[【SpringBoot】监听器篇](http://zhangh.tk/2017/07/05/%E3%80%90SpringBoot%E3%80%91%E7%9B%91%E5%90%AC%E5%99%A8%E7%AF%87/)

自定义事件、事件监听器的使用：

- [Spring Boot 自定义事件监听器](https://github.com/zhanghTK/spring-boot-start/tree/master/listener)
- [Spring 自定义事件监听器](https://github.com/zhanghTK/spring-boot-start/tree/master/refresh)

### 事件广播器

![事件广播器.png](https://i.loli.net/2017/08/14/599114f2dda29.png)

Spring 事件广播器的初始化：[Spring 事件广播器的初始化](http://zhangh.tk/2017/07/16/%E3%80%90Spring%E3%80%91%E5%AE%B9%E5%99%A8%E5%88%B7%E6%96%B0/#initMessageSource)

#### 事件同步、异步广播

事件广播：

```java
@Override
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
   ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
   for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
      Executor executor = getTaskExecutor();
      if (executor != null) {
         executor.execute(new Runnable() {
            @Override
            public void run() {
               invokeListener(listener, event);
            }
         });
      }
      else {
         invokeListener(listener, event);
      }
   }
}
```

方法一：指定线程池

当未指定线程池时（getTaskExecutor = null），使用同步的方式广播线程，因此可以同步指定线程池开启异步调用。缺点：全局异步，无法对指定的事件进行异步处理。

方法二：@Aync 注解

使用 @Aync 注解指定需要异步调用的事件监听器监听器。
