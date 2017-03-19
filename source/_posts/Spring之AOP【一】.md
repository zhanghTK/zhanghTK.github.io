---
title: Spring之AOP【一】
date: 2017-03-20 02:33:06
tags:
  - Java
  - Spring
categories: Spring
---

## 关于AOP

实现方式：

- 动态代理【Spring AOP默认】
- 动态字节码增强【Spring AOP备选】
- Java代码生成
- 自定义类加载器
- AOP扩展

AOP组成：

- Joinpoint：添加（织入）横切逻辑的位置
- Pointcut：Jointpoint的表述方式
- Advice：横切逻辑
- Weaver：织入器
- 目标对象

## Spring AOP

Spring AOP实现机制：动态代理机制和字节码生成技术实现，在运行期为目标对象生成一个代理对象。

Spring对AOP的实现实质上是使用代理机制对上述基本AOP组成概念实现，组合。

### Pointcut

Spring AOP只支持方法级别的Joinpoint，在Pointcut上也可以体现出来：Pointcut接口只定义两个方法分别用于匹配类和方法。

Pointcut根据方法匹配可以分为两类：

- StaticMethodMatcherPontcut，不会检查Joinpoint的方法参数
- DynamicMethodMatcherPointcut，每次都要对方法参数进行检查

具体的实现可以不做任何限制，可以根据方法名，注解等各种形式，正则，逻辑运算，调用顺序等多种方式进行过滤。

### Advice

跟与Advice是否在目标对象之间共享可以分为：per-class和per-instance

#### per-class

Advice实例可以在目标对象的所有实例之间共享，又可以分为：

- Before Advice
- ThrowsAdvice
- AfterReturningAdvice
- AroundAdvice

每种情况都有对应有接口，要实现对应的Advice之需要实现对应接口。

AroundAdvice比较特殊，Spring没有定义对应接口，而是使用了AOP Alliance的标准接口，接口定义如下：

```java
public interface MethodInterceptor extends Interceptor {
  Object invoke(MethodInvocation invocation) throws Throwable;
}
```

MethodInterceptor：Advice，唯一方法invoke封装横切逻辑

MethodInvocation：控制拦截行为，可以获取Joinpoint的信息，重要方法：process

#### per-instance

Advice会为不同实例对象保存各自的状态及逻辑，Spring围绕Introduction实现per-instance型Advice。

![Introduction.png](https://ooo.0o0.ooo/2017/03/19/58ce9fe4f321a.png)

拦截器——IntroductionInterceptor

继承自：

- DynamicIntroductionAdvice：判断给定接口是否是扩展逻辑
- MethodInterceptor：
  - 当接口是扩展逻辑：通过Method.invoke()执行扩展逻辑
  - 当接口不是扩展逻辑：通过MethodInvocation.proceed()调用代理对象

Introduction可以分为两类：

- 以IntroductionInfo为首的静态分支
  - 预先定义扩展逻辑接口
  - DelegatingIntroductionInterceptor：内部持有一个扩展逻辑实现类，供统一目标类的所有实例共享使用
- DynamicIntroductionAdvice为首的动态分支
  - 运行时获取扩展逻辑接口
  - DelegatePerTargetObjectIntroductionInterceptor：内部持有一个映射关系，映射目标对象与扩展逻辑实现类

### Aspect

在Spring第一代的实现中是没有Aspect的概念的，与之对应的是Advisor。

根据Advice的不同，又可以分为：PointcutAdvisor和IntroductionAdvisor，其中IntroductionAdvisor是专门为Introduction使用的。

当在一个Joinpoint处理多个Advice时，可以指定优先级。

#### PointcutAdvisor

根据Pointcut，Advisor的不同实现，PointcutAdvisor提供了多种具体实现组合

####  IntroductionAdvisor

只有一个默认实现：DefaultIntroductionAdvisor

### Weaver

通过Aspect封装了Pointcut和Advice之后，Weaver只需要关注目标对象和Aspect就可以了。

根据目标对象和Advice，织入器就可以完成织入，典型的织入器有：ProxyFactory，ProxyFactoryBean

#### ProxyFactory

Spring最基本的织入器——ProxyFactory：

![ProxyFactory.png](https://ooo.0o0.ooo/2017/03/20/58cec4ff8c8fd.png)

AdvicedSupport包含生成代理对象所需的信息

AopProxy抽象不同AOP实现（动态代理，CGLIB）

最终对ProxyFactory进行配置后就可以生成代理对象了，配置功能是从ProxyCreatorSupport继承而来，生成代理对象委托给具体的AopProxy实现类。

#### ProxyFactoryBean

与ProxyFactory功能类似，本质是一个产生代理对象的FactoryBean

#### 自动织入

无论ProxyFactory还是ProxyFactoryBean都需要配置目标对象和Advice才能完成一个织入过程，自动织入可以借助IoC容器，在BeanPostProcessor阶段自动扫描完成所有织入动作。

整个自动织入过程可以描述为：

```java
for bean in IoC container
  if bean isAutoProxyBean
    return createProxy(bean)
  else
    return createInstance(bean)
```

Spring默认提供了多个可用的AutoProxyCreator，可以根据bean name指定目标对象，扫描Advice生成代理对象，甚至可以全自动生成代理对象。


### TargetSource

作为目标对象的容器，TargetSource最重要的作用就是获取目标对象。

TargetSource不同的实现提供了目标对象singleton，prototype，热替换，对象池等多种管理形式。

在多数据源替换的场景还是很有用的。
