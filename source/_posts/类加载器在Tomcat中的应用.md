---
title: 类加载器在Tomcat中的应用
date: 2017-03-13 22:22:36
tags:
  - Java
  - JVM
categories: Java
---

## 类加载器

关于类加载器的文章有很多了，概括起来有如下几点：

- 双亲委派模型
  - 自底向下检查类是否已加载
  - 自顶向下尝试加载类
- 上下文类加载器：解决顶层类加载器无法加载底层类加载器加载的类（这句话好绕口）
- 其他容易忽视的细节问题：
  - 一个类的加载器可能不止一个
    - 不同类加载器加载同一个类
    - 同一个类加载器不同实例加载同一个类
  - 完成一次加载过程的类加载器也可能不止一个
    - 初始加载器：启动类的加载过程
    - 定义加载器：真正完成类的加载工作
  - 系统或扩展路径下的class文件会优先被系统类加载器或扩展类加载器加载
  - loadClass方法抛出的是ClassNotFoundException
  - defineClass方法抛出的是NoClassDefFoundError
  - 在不指定父类加载器的情况下，默认采用系统类加载器

## Tomcat

在Tomcat中对类加载器的要求稍微复杂一些：

- 不同web app下不同的类要做到隔离，不能互相干扰
- 不同web app下相同的类要可以共享，避免重复加载

以上两个功能根据两个类加载就可以实现：

- WebappClassLoader
- CommonClassLoader

Tomcat类加载器与Java自带类加载器组成的关系结构：

![Tomcat类加载器架构.png](https://ooo.0o0.ooo/2017/03/13/58c6ae1596e2d.png)

### WebappLoader

默认的WebappClassLoader主要由以下特点：
- 加载/WebApp/WEB-INF/*下的内容

- 每个web app对应一个WebappLoader

  确保不同web app的类由不同的WebappLoader实例加载

- 不遵循双亲委派

  优先使用WebappClassLoader自己加载，而不是委托给父类加载确保隔离

- 避免Servlet API，java.， javax.相关类加载

### CommonClassLoader
- 加载/common/*下的内容

- 统一加载容器和各个web app都需要的类

- 在容器启动阶段就将CommonClassLoader设置为上下文类加载器

- 所有WebappLoader的parent都是CommonClassLoader

### 联系
- CommonClassLoader能加载的类都能够被CatalinaClassLoader和WebappClassLoader使用；

- 各个WebappClassLoader实例之间互相隔离

- JasperLoader加载范围仅限JSP文件编译出的那个Class，方便丢弃实现热替换
