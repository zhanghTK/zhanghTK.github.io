---
title: Spring事务行为
date: 2017-12-08 21:15:53
tags:
  - Java
  - Spring
categories: Spring
---
## 事务

### 特性

原子性：事务是一个不可分割的单位，事务中的操作要么都发生，要么都不发生。

一致性：事务执行前后数据的完整性必须保持一致

隔离性：多个用户并发访问数据库时，一个用户的事务不能被其他用户的事务所干扰，多个并发事务之间数据要相互隔离

持久性：一个事务一旦被提交，它对数据库中的数据的改变就是永久性的，即使数据库发生故障也不应该对其有任何影响

### 隔离级别

隔离级别引发的问题：

- 脏读：一个事务读取到了另一个事务改写但还未提交的数据（如果这些数据被回滚，则读到的数据是无效的）
- 不可重复读：在同一个事务里，多次读取同一数据返回的结果有所不同
- 幻读：一个事务读取了几行记录后，另一个事务插入（或删除）了一些记录，第一个事务再查询发生不一致

隔离级别：

| 隔离级别            | 含义                        | 脏读   | 不可重复读 | 幻读   | 备注        |
| --------------- | ------------------------- | ---- | ----- | ---- | --------- |
| READ_UBCOMMITED | 允许读还未提交的改变了的数据            | 是    | 是     | 是    |           |
| READ_COMMITED   | 允许在并发事务应景提交后读取            | 否    | 是     | 是    | Oracle 默认 |
| REREATABLE_READ | 对相同字段的多次读取是一致的，除非数据被本事务改变 | 否    | 否     | 是    | MySQL默认   |
| SERIALIZABLE    | 完全服从 ACID 的隔离级别           | 否    | 否     | 否    |           |

## Spring 接口

Spring 事务管理高层抽象主要的接口有：

- 事务管理器：PlatformTranscationManager
- 事务定义信息：TransactionDefinition
- 事务具体运行状态：TransactionStatus

使用 TransactionDefinition 定义事务信息，由 PlatformTransactionManager 负责执行事务，执行的结果记录在 TransactionStatus。

### PlatformTransactionManager

包含多个实现，可以为不同持久化框架提供不同实现

| 实现                           | 说明                                  |
| ---------------------------- | ----------------------------------- |
| DataSourceTransactionManager | 使用 Spring JDBC 或 MyBatis 进行持久化数据时使用 |
| HibernateTransactionManager  | 使用 Hibernate 进行持久化数据时使用             |
| JpaTransactionManager        | 使用 JPA 进行持久化时使用                     |
| JdoTransactionManager        | Jdo 持久化机制时使用                        |
| JtaTransactionManager        | 使用 JTA 管理事务                         |

### TransactionDefinition

- 常量

  ISOLATION_XXX 定义了事务的隔离级别

  PROPAGATION_XXX 定义了事务的传播行为

  TIMEOUT_DEFAULT 默认超时

- 方法

  获得事务以上信息

#### 隔离级别

Spring 事务隔离级别所有事务隔离级别，默认使用 DB 的事务隔离级别

#### 传播行为

传播行为解决事务方法相互调用时，事务的处理方式。Spring 事务提供的传播行为：

| 事务传播行为                    | 含义                      |
| ------------------------- | ----------------------- |
| PROPAGATION_REQUIRED      | 支持当前事务，如果不存在就新建一个       |
| PROPAGATION_SUPPORTS      | 支持当前事务，如果不存在，就不使用事务     |
| PROPAGATION_MANDATORY     | 支持当前事务，如果不存在，抛出异常       |
| PROPAGATION_REQUIRES_NEW  | 如果有事务存在，挂起当前事务，创建一个新的事务 |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式运行，如果有事务存在，挂起当前事务 |
| PROPAGATION_NEVER         | 以非事务方式运行，如果有事务存在，抛出异常   |
| PROPAGATION_NESTED        | 如果当前事务存在，则嵌套事务（保存点）     |

##### PROPAGATION_REQUIRED

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
  methodB();
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() {
}

methodB();  // 当前上下文不存在事务，methodB 开启一个新的事务
methodA();  // 当前上下文不存在事务，methodA 开启一个新的事务，当执行内部 methodB() 时，methodB 发现当前上下文有事务，因此就加入到当前事务中来
```

##### PROPAGATION_SUPPORTS

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
  methodB();
}

@Transactional(propagation = Propagation.SUPPORTS)
public void methodB() {
}

methodB();  // methodB 以非事务的方式执行
mtthodA();  // 当前上下文不存在事务，methodA 开启一个新的事务，当执行内部 methodB() 时，methodB 加入 methodA 的事务
```

##### PROPAGATION_MANDATORY

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
doSomeThingA();
  methodB();
}


// 事务属性为REQUIRES_NEW
@Transactional(propagation = Propagation.MANDATOR)
public void methodB() {
}

methodB();  // 当前没有一个活动的事务，则会抛出异常throw new IllegalTransactionStateException(“Transaction propagation ‘mandatory’ but no existing transaction found”);
methodA();  // 当前上下文不存在事务，methodA 开启一个新的事务，当执行内部 methodB() 时，methodB 加入 methodA 的事务
```

##### PROPAGATION_REQUIRES_NEW

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
  doSomeThing1();
  methodB();
  doSomeThing2();
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void methodB() {
}
```

当调用 methodB() ，当前上下文不存在事务，methodB 开启一个新的事务

当调用 methodA() 相当于执行了以下的代码：

```java

TransactionManager tm = null;
try{
  // 获得一个JTA事务管理器
  tm = getTransactionManager();
  tm.begin();// 开启一个新的事务
  Transaction ts1 = tm.getTransaction();
  doSomeThing1();
  tm.suspend1();// 挂起当前事务
  try{
    tm.begin();// 重新开启第二个事务
    Transaction ts2 = tm.getTransaction();
    methodB();
    ts2.commit();// 提交第二个事务
  } Catch(RunTimeException ex) {
    ts2.rollback();// 回滚第二个事务
  } finally {
    // 释放资源
  }
  // methodB执行完后，恢复第一个事务
  tm.resume(ts1);
  doSomeThing2();
  ts1.commit();// 提交第一个事务
} catch(RunTimeException ex) {
  ts1.rollback();// 回滚第一个事务
} finally {
  // 释放资源
}
```

上面的 ts1 和 ts2 是两个独立的事务，互不干扰， ts2 是否成功并不依赖于 ts1。如果 methodA 在调用 methodB 方法后的 doSomeThing2 发生异常，methodB 并不受影响结构依然没提交，但 methodA 的其他代码则会被回滚。

使用 PROPAGATION_REQUIRES_NEW 需要使用 JtaTransactionManager 作为事务管理器。

##### PROPAGATION_NOT_SUPPORTED

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
  doSomeThing1();
  methodB();
  doSomeThing2();
}

@Transactional(propagation = Propagation.PROPAGATION_NOT_SUPPORTED)
public void methodB() {
}
```

总是以非事务的形式执行，当 methodA 执行到 methodB(); 时先挂起事务，再执行 methodB(), 完成后再恢复 methodA 的事务继续执行 doSomeThing2 方法。

使用PROPAGATION_NOT_SUPPORTED,也需要使用JtaTransactionManager作为事务管理器。

##### PROPAGATION_NEVER

总是非事务地执行，如果存在一个活动事务，则抛出异常。

##### PROPAGATION_NESTED

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA(){
  doSomeThing1();
  methodB();
  doSomeThing2();
}

@Transactional(propagation = Propagation.NEWSTED)
public void methodB(){
}
```

当调用 methodB() ，则按照 PROPAGATION_REQUIRED 执行，当前上下文不存在事务，methodB 开启一个新的事务

当调用 methodA() 相当于执行了以下的代码：

```java
Connection con = null;
Savepoint savepoint = null;
try{
  con = getConnection();
  con.setAutoCommit(false);
  doSomeThing1();
  savepoint = con2.setSavepoint();
  try{
    methodB();
  } catch(RuntimeException ex) {
    con.rollback(savepoint);
  } finally {
    //释放资源
  }
  doSomeThing2();
  con.commit();
} catch(RuntimeException ex) {
  con.rollback();
} finally {
  //释放资源
}
```

在调用 methodB 之前，先调用 setSavepoint 方法，保存当前的状态到 savepoint。如果 methodB 执行失败，则恢复到  savepoint 保存的状态。

如果外层事务执行失败，会回滚内层事务所做的动作；

如果内层事务执行失败，不会引起外层事务的回滚；

使用JDBC 3.0驱动时,仅仅支持 DataSourceTransactionManager 作为事务管理器，PlatformTransactionManager的nestedTransactionAllowed属性设为true(属性值默认为false)

### TransactionStatus

维护，获取事务的各种状态

## 使用 Spring 事务

Spring 提供编程式和声明式的事务管理，具体的代码实现可以参考：[spring-tx-test](https://github.com/zhanghTK/spring-tx-test)
