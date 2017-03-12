---
title: Spring之BeanFactory
date: 2017-03-13 01:19:54
tags:
  - Java
  - Spring
categories: Spring
---
Spring核心的功能是IoC和AOP，IoC又是AOP的基础。对于IoC，Spring提供了两种容器类型：

- BeanFactory
- ApplicationContext

ApplicationContext可以简单理解为是BeanFactory的升级版。

本文试图讨论与BeanFactory相关的大致过程。

### 调用实现

#### 直接编码

BeanFactory是基础的IoC容器，提供了完成的IoC服务。先看看BeanFactory和Bean之间的关系：

![BeanFactory.png](https://ooo.0o0.ooo/2017/03/12/58c4fc455dda9.png)

BeanFactory，BeanDefinitionRegistry，BeanDefinition都是基本接口。其中：

- BeanFactory定义了基本的查询相关方法；
- BeanDefinitionRegistry定义了Bean注册管理的相关方法；
- BeanDefinition定义容器中的一个Bean实例；

DefaultListableBeanFactory实现了Bean注册，查询相关方法，Bean之间的关系则通过Bean实例维护，以上构成了一个最基本的IoC结构。

使用Spring IoC最直接的方式就是依赖上述几个类来加载、维护、查询Bean实例，只是当更先进的方式到来后，直接编码的方式已经离我们很远了。

#### 外部配置

程序员总是希望以更便捷的方式维护配置信息，配置文件是其中一个手段，也是Spring配置最常使用的一种方式。

Spring配置文件最常见的方式是XML形式，其实还有Properties，得益于良好可扩展性，我们甚至可以自定义配置文件及加载方式。

BeanDefinitionReader通过处理外部配置文件，根据不同的配置文件格式，BeanDefinitionReader不同子类将相应的配置文件内容加载并解析映射到BeanDefinition，然后将映射后的BeanDefinition测试到BeanDefinitionRegistry。

加入BeanDefinitionRegistry后的类图：

![BeanFactory.png](https://ooo.0o0.ooo/2017/03/12/58c5115a0c0fa.png)

### 背后的细节

IoC容器功能实现简单可以分类两个阶段：

- 容器启动
- Bean实例化

#### 容器启动

容器启动主要的过程包括了：

1. 加载配置
2. 分析配置信息
3. 装配到BeanDefinition
4. 其他后续

在整个启动阶段可以通过BeanFactoryPostProcessor，在实例化阶段开始之前，对注册到容器的BeanDefinition保存的原始数据做出修改。

Spring自带了几个BeanFactoryPostProcessor的实现：

- PropertyPlaceholderConfigurer：占位符替换
- PropertyOverrideConfigurer：替换bean字段
- CustomEidtorConfigurer：配置解析

#### Bean实例化

对于BeanFactory，当容器启动后只有当客户端调用`getBean()`方法时才会触发实例化阶段活动。

完整的Bean实例化过程如下：

![Bean实例化.png](https://ooo.0o0.ooo/2017/03/12/58c51014ca1e0.png)

几个说明的点：

1. Bean的实例化

   实例化有两种方式实现：

   - 反射
   - CGLIB（默认）

   实例化完成后，不直接返回生成的实例化对象，使用BeanWrapper对对象进行包裹

2. 设置对象属性

   BeanWrapper使用PropertyEditor对获取的属性值做出相应转换，设置对象属性值

3. 设置Aware依赖

   容器根据Aware接口，对Bean实例设置相应属性

4. BeanPostProcessor

   一般用于筛选bean，对bean实例化过程扩展，AOP更多在此生成代理对象

   ApplicationContext在此阶段设置Aware依赖
