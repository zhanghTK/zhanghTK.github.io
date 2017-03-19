---
title: Spring之ApplicationContext
date: 2017-03-19 20:57:27
tags:
  - Java
  - Spring
categories: Spring
---
相较于BeanFactory，ApplicationContext进一步扩展了基本功能。主要有：统一资源加载，国际化信息支持，容器内部事件发布，多配置模块加载的简化。

## 统一资源加载

在Spring中资源的表示和加载时分开的：使用Resource表示资源，使用Resource Loader加载资源。

二者与Application Context的关系：

![ResourceLoader.png](https://ooo.0o0.ooo/2017/03/19/58ce56622ec9d.png)

从很多地方都能看出Spring优秀的设计，资源加载策略就是其中。

为什么说统一的资源加载？因为什么资源都能加载。

首先，Spring抽象了资源的表示以Resource表示各种资源的形式，根据具体形式又提供了例如ClassPathResource的具体形式。

其次，ResourceLoader抽象了资源查找定位策略的抽象，其分为大体可以分为两类：

- 以DefaultResourceLoader为基础的，一般的ResourceLoader
- 以ResourcePatternResolver为基础的，批量查找的ResourceLoader

容器启动是加载的配置文件所使用的资源路径协议也是在此定义的。

最后看看Application Context与Resource Loader的关系：Application Context是一个Resource Loader，更一般的情况Application Context还是ResourcePatternResolver

## 国际化信息支持

Spring对国际化得支持不仅是针对还是对整个应用的支持，以MessageSource作为国际化信息的访问接口，封装了相应的信息查询，针对不同的需求做出了不同实现（例如热替换）。

Application Context同时继承了MessageSource，因此容器获得国际化的支持，对普通Bean可以通过注入MessageSource获得国际化的能力。

## 容器内部事件发布

标准的事件发布功能包含三个角色：自定义事件类型，事件监听接口，事件发布者。Spring对相关角色的定义：

![事件发布.png](https://ooo.0o0.ooo/2017/03/19/58ce7c99afcc3.png)

Application Context继承ApplicationEventPublisher，将具体的实现委托给ApplicationEventMulticaster接口的实现类来实现。

具体的ApplicationEvent实现Spring提供了容器生命周期相关的和Web请求相关的事件类型，容器还支持自定义事件类型的发布。

## 多配置模块加载的简化

相较于BeanFactory，Application Context还提供多个配置文件加载的简化

## Application Context

一言以蔽之，Application Context是这样的Bena容器：

![Application Context.png](https://ooo.0o0.ooo/2017/03/19/58ce7f9a578b6.png)
