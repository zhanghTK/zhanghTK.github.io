---
title: Spring之数据访问
date: 2017-04-16 16:19:33
tags:
  - Java
  - Spring
categories: Spring
---

提起JDBC的使用大家一般集中喷两个地方：

1. JDBC的异常处理
2. JDBC数据处理的冗长代码

看看Spring中对这两个问题是怎么举得的吧：

## 统一的数据访问异常层次体系

针对数据访问异常JDBC定义了4个异常，可以说是高度的抽象了，但换句话说就是没什么卵用。

尤其是SQLException，这个异常是个checked exception。如果直接抛出会污染上层接口；如果在数据访问层内处理，这个异常却包含了多种问题，例如：

- 无法连接数据库
- SQL语法错误
- SQL违反数据库约束

即使拿到SQLException我们并不能马上准确的获取具体异常问题。

Spring解决的也是简单直接：

- 所有的数据访问异常都定义为了unchecked exception。你嫌弃它要抛出，那就都不要抛出了
- 转译SQLException。你说它不知道怎么处理，那就提供更细粒度的异常

## 数据访问模板——JdbcTemplate

提到数据访问模板我想起了在学习JDBC操作时，当时写过一个标准的JDBC增删查该的模板，整个过程涉及到的内容还是不那么简单的。

Spring在这里直接提供了JdbcTemplate作为数据访问的基础，这个类主要解决了两件事：

1. 使用模板模式封装基于JDBC的数据访问
2. 对数据访问异常转译，将JDBC异常转化为Spring自定义数据访问异常

针对Jdbc的操作和Jdbc的访问Spring分类提供了两个抽象接口由JdbcTemplate继承：

![JdbcTemplate.png](https://ooo.0o0.ooo/2017/04/16/58f3143a23751.png)

JdbcAccessor：JdbcTemplate的直接父类，主要两个属性：

- DataSource：作为Jdbc的连接工厂
- SQLExceptionTranslator：处理SQLException的转译

JdbcOperations：定义JDBC操作集合的抽象接口。

针对不同类型的操作，Spring提供不同的回调接口而不是简单的抽象模板

除此，JdbcTemplate内部实现细节还有值得注意的地方：

**connection**

connection有两点需要注意：

1. connection获取是使用DataSourceUtils获取的。而不是简单的从dataSource中获取的。

   DataSourceUtils会将connection绑定到当前线程，以便事务管理使用

2. 具体的connection由NativeJdbcExtractor获取

**查询控制**

具体的查询语句执行前会先调用applyStatementSettings方法以控制具体的查询行为。

**异常转译**

SQLExceptionTranslator作为异常转译的接口，其具体可以针对message，errorCode，自定义信息转译SQLException。

**JdbcTemplate的增强**

除了基本的JdbcTemplate之外，Spring还提供了NamedParameterJdbcTemplate等操作以增强JdbcTemplate的基本功能。

**DataSource**

Spring提供了多个DataSource实现，也可以使用第三方的，自定义DataSource既可以继承AbstractDataSource或者实现DelegatingDataSource。

这里额外提及TransactionAwareDateSourceProxy，从其获取的Connection可以自动加入Spring的统一事务管理。
