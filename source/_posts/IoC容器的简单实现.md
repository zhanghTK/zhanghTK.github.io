---
title: IoC容器的简单实现
date: 2016-11-10 23:44:15
tags:
  - Java
categories: Java

---

记录临摹一个IoC容器的过程，使用对象容器进行控制反转，将对象间依赖关系的管理交给容器。

代码在[这里](https://github.com/zhanghTK/HelloIoC)，API参照了Spring IoC部分，实现的具体过程参照了[tiny-spring](https://github.com/code4craft/tiny-spring)和[ioc-sample](https://github.com/kevinlynx/ioc-sample)。先看看如何使用：

# 像Spring一样用

```xml
<beans>
    <bean name="helloWorldOutputService"
          class="tk.zhangh.ioc.beans.HelloWorldOutputServiceImpl">
        <property name="name" value="helloWorld"/>
        <property name="outputService" ref="outputService"/>
    </bean>
</beans>
```

```java
@Test
public void register_get_bean_by_ClassPathXmlApplicationContextTest() throws Exception {
    ApplicationContext applicationContext = new
      ClassPathXmlApplicationContext("ioc.xml");
    HelloWorldService helloWorldService = (HelloWorldService)
      applicationContext.getBean("helloWorldOutputService");
    helloWorldService.sayHello();
}
```

---

整个开发过程是这样的：

# 自下而上

大体的开发步骤以及思路参照了[tiny-spring](https://github.com/code4craft/tiny-spring)，实现步骤可以简述为：

## 1.全手动式的Bean容器

做容器的最重要的就是存取，针对bean容器就是bean信息的保存（注册）和bean实例的获取。

- bean信息注册

  bean的基本信息包括：bean的名称、bean实例、bean的Class信息，bean的属性信息，把这些基本信息封装成了`BeanDefinition`。

  以`key(beanName)=>value(BeanDefinition)`的键值对就可以完成注册的功能

- bean实例获取

  从bean信息到bean实例中间还有一条鸿沟：怎么实例化bean。

  为了方便实现，在初始化`BeanDefinition`实例的时候也对bean进行了初始化。


站在客户端角度，怎么注册bean并不重要，因此bean容器接口只声明获取bena的方法。

在`BeanDefinition`中实例化bean带来一个严重问题：bean实例的创建不受容器控制的。

## 2. 定制Bean的实例化过程

针对上面的问题，`AbstractBeanFactory`抽象出实例化bean方法，并在`AutowireCapableBeanFactory`提供基板实现。在`BeanDefinition`中并不需要再实例化了（代码实现到这一步时没有修改,bug）。

新的`AutowireCapableBeanFactory`已经可以做到：

1. 注册保存`BeanDefinition`
2. 在注册时实例化bean
3. 提供bean实例的获取

现在，bean的实例化是受控于容器的。

但是初始化的时机不够灵活，整个生命周期只有在注册时刻有唯一一次初始化。

这样会影响bean实例属性的初始化，先看基本属性：

## 3. 支持基本属性依赖

对基本属性支持比较简单，整个过程完全由容器控制：

1. 根据`BeanDefinition`获取bean相关的属性信息
2. 创建对应的属性对象
3. 使用反射注入属性

在bean实例构造完成后就对属性注入，但是现有的方案并不能支持bean属性的注入：

1. 属性bean从哪里来
2. bean属性本身依赖其他bean呢？如果存在循环依赖呢？

## 4. 使用资源文件配置

解决bean属性问题前，先完成支持资源文件的管理。

1. 创建资源文件表示类，以及资源加载类
2. 创建信息读取接口，抽象类，以及具体的XML配置读取策略类

添加对资源文件配置支持后，整个bean容器的过程为：

1. 读取加载配置文件信息
2. 创建beanfactory
3. 注册保存bean信息
   1. 创建bean实例
   2. 设置bean实例的属性

支持资源文件配置后原来需要硬编码的bean信息可以以配置文件的形式展示。

但是，bean的实例构造时机、属性注入的时机没有改变，所以依赖存在对bean属性的支持问题。

## 5. 支持bean属性依赖

前面碰到的两个问题：

1. 属性bean从哪里来
2. bean属性本身依赖其他bean呢？如果存在循环依赖呢？

属性bean也是bean，所以应该从工厂获取，循环依赖问题在Spring里是通过延迟实例化解决。

所以问题变成了怎么调整bean实例构造时机，让这个过程延迟，发生在所有bean注册完成后。

调整：在注册时只保存`BeanDefinition`，不对bena进行实例化。

所有bean的实例化延迟到第一次获取bean实例时再进行：

1. 先创建bean实例
2. 遍历所有属性，基本属性直接注入，如果是其他bean引用重复以上过程。

## 6. 进一步简化

回头看现在的客户端使用：

1. 加载资源
2. 解析资源
3. 创建beanFactory
4. 注册bean
5. 获取bean

前四步实际是beanFactory的生命周期内容，客户端不关心这些细节，只要提供配置文件就可以。

所以使用`ApplicationContext`接口对外暴露获取bean的方法。

bean加载，解析，存取功能分别委托给：`BeanDefinitionReader`， `AbstractBeanFactory`。

整个Bean容器生命周期细节都可以封装起来，对外提供简单调用。



至此，一个简单的IoC容器就完成了

---

# 自上而下

俯视这个IoC容器，基本的生命周期活动包括了：

1. 资源加载
2. 资源解析
3. bean factory创建
4. bean注册
5. 创建bean实例
6. bean获取

所有的步骤都是以接口或抽象方法的形式提供或者是多态留白，具体的实现都交由子类实现。

|       步骤       |                   抽象方法                   |
| :------------: | :--------------------------------------: |
|      资源加载      |        Resources.getInputStream()        |
|      资源解析      | BeanDefinitionReader.loadBeanDefinitions(String) |
| bean factory创建 |      this.beanFactory = beanFactory      |
|     bean注册     | AbstractBeanFactory.registerBeanDefinition(String, BeanDefinition) |
|    bean实例化     | AbstractBeanFactory.doCreateBean(BeanDefinition) |
|     bean获取     |       BeanFactory.getBean(String)        |

各个模块的耦合以接口留白的形式而非具体实现类为扩展带来了极大的灵活。

在实现的时候并没有考虑这些，但实现完成后发现接口带来的灵活极大的方便了修改和扩展。
