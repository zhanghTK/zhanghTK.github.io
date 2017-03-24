---
title: Spring之AOP【二】
date: 2017-03-25 02:42:16
tags:
  - Java
  - Spring
categories: Spring
---

## @AspectJ形式的Pointcut

### Pointcut的组成：

- Pointcut Expression
  - 以@Pointcut为载体
  - 由两部分组成：Pointcut标识符，表达式匹配模式
- Pointcut Signature
  - Pointcut Expression的载体

#### Pointcut标识符

**execution**

日常使用最多的标识符，使用execution标识符的Pointcut表达式格式：

```
execution (modifiers-pattern? ret-type-pattern declaring-type-pattern? 
			name-pattern(param-pattern) throws-pattern?)
```

- 方法返回类型、方法名以及参数必须制定，其他可以省略
- 可选通配符：`*` 和` ..`
  - `*`：匹配一个单词
  - `..`：只能在declaring-type-pattern和param-pattern位置使用
    - 用于declaring-type-pattern可以指定多个层次
    - 用于param-pattern表示可以有0到多个参数，可以与`*`和具体类型组合

**within**

指定类型，类型下所有方法。可以使用`*`和`..`扩展，like：`within(tk.zhangh.spring..*)`

**this和target**

- this：目标对象的代理对象
- target：目标对象

Spring中使用this和target实际作用类似

**args**

指定参数类型，指定参数数量

与execution标识符不同，args标识符会在运行期间动态检查参数类型

**@within**

指定类型，类型下的所有方法，要求类型标记了指定注解，like：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD, ElementType.TYPE})
public @Log AnyJoinpontAnnotation{}

@Log
public class Bean {
  public void method();
}

@with(Log)
```

**@target**

指定标记了给定注解类型的目标对象的所有方法

**@args**

指定参数类型，要求参数参数类型标记了指定注解

**@annotation**

指定标记了指定注解的方法，@Transctional的实现方式

### Pointcut表达式的解析

所有@AspectJ形式声明的这些Pointcut表达式最终都会转化成Pointcut的具体实现。

AspectJExpressionPointcut如同他的名字面向AspectJ的pointcut实现，整个继承体系：

![Pointcut.png](https://ooo.0o0.ooo/2017/03/23/58d3d30a2cc93.png)

## @AspectJ形式的Advice

使用@Aspect注解标记的类中，具体的Advice形式由具体的Advice注解标示。

注解的方法中需要访问上下文信息最主要的方式：将方法的第一个参数声明为JoinPoint类型

- @Before

- @AfterReturning

- @AfterThrowing

  - 获取异常：在JoinPoint类型参数后面加上RuntimeException类型参数

- @After

- @Around

  - 获取上下文信息不同以上，需要方法第一个参数声明为ProcessingJoinPoint类型

- @DeclareParents

  - 最特殊，使用如下：

  ```java
  @Aspect
  public class IntroductionAspect {
    @DeclareParents(value="....NewImpl", defaultImpl=Target.class)
    public INew new;
  }
  ```

## 其他

### Advice执行顺序

- 同一个Aspect中最先声明的Advice拥有最高优先级，优先入栈
- 不同Aspect的Advice需要实现Order接口声明优先级

### 同一对象内的嵌套方法调用拦截失效

以事务为例，事务管理也是使用AOP，具体是@annotation形式的Pointcut声明（这样我就不用声明Advice了）

```java
public class ServiceImpl implents Service {
  public void methodA() {
    doSomething();
    methodB();
  }
  
  @Transactional
  public void methodB() {
    // ...
  }
}
```

当在aFun内调用bFun时事务没有开启，也就是AOP没有生效，原因：

![AOP嵌套调用.png](https://ooo.0o0.ooo/2017/03/24/58d5043b70bbc.png)

我们期望虚线的调用方式，实际上调用时红色的路线，添加在代理对象上的AOP逻辑在嵌套调用时根本没有机会触发。在事务处理时尤其要注意避免这样的嵌套调用问题。

解决：

- 使用AopContext.currentProxy()获取代理对象
- 从ApplicationContext中获取代理对象

不管是那种方式都要注入相关Bean，具体那种更优雅由你来决定了。
