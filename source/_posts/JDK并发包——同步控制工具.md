---
title: JDK并发包——同步控制工具
date: 2017-04-09 21:46:11
tags:
  - Java
  - 并发
categories: Java
---

为了弥补Java原生并发的各种不足，在并发包中提供了格式各样的工具，这篇先看看关于同步控制并法包的支持。

## 可重入锁——ReentrantLock

Java原生提供的synchronized主要有以下几个缺点：

- JDK5之前的synchronized的性能较差
- synchronized本身无法响应中断、无法限时等待、无法确保同步线程之间的公平性

可重入锁既是为了解决以上问题提出的。

但是关于这个可重入的名字我觉得还是比较让人疑惑，synchronized本身也是支持一个线程的重入的。

对于原生synchronized的缺陷，ReentrantLock提供了如下几个重要的方法以改善：

- lockInterruptibly()：获得锁，但优先响应中断
- tryLock()：尝试获得锁，成功返回true，失败立即返false，不等待
- tryLock(long time, TimeUnit unit)：在给定时间内尝试获得锁
- unlock()：释放锁
- ReentrantLock(boolean fair)：确保同步线程之间的访问公平

除此还提供了最基本的lock()方法：获得，如果锁被占用则等待。

ReentrantLock使用过程中需要注意的是无论何种方式获取锁，每次获取锁之后都要手动调用unlock方法释放锁。

## wait/notify的替身——Condition

Condition与ReentrantLock的关系如同synchronized与wait/notify方法的关系。

为了配合ReentrantLock，Condition本身提供了与wait方法类似的await，与notif方法类似的singal，除此还补充了不响应中断的awaitUninterruptibly方法。

关于Condition的使用：

- 通过ReentrantLock实例的工厂方法获取Condition。


- 调用await时必须持有线程的相关ReentrantLock，await调用后线程会释放锁，当再次被唤醒时重新持有锁
- 调用signal时也必须持有线程的相关ReentrantLock，但是调用结束后需要手动释放锁，否则被唤醒线程无法获取锁

## 宽容的临界区——Semaphore

可以简单的理解信号量（Semaphore）是可以多个线程同时访问的临界区资源。

在构造Semapho时必须制定多个线程的具体数量，还可以指定是否是否公平访问，对象的其它主要方法有：

- acquire()
- acquireUninterruptibly()
- tryAcquire()
- tryAcquire(long timeout, TimeUnit unit)
- release()

方法的含义如同方法名，没有发现使用特别的地方。

需要注意的是申请的信号量使用完毕后，需要release以避免资源越来越少。

## 读写锁——ReentrantReadWriteLock

说来惭愧，到目前为止在实际项目中并发包的各种工具我貌似只使用过读写锁。

通常锁都是严格串行的，读写锁提供读写分离锁，对多个读操作不进行阻塞，以此来改善性能。

具体使用中读锁和写锁需要从同一个ReentrantReadWriteLock对象获取：

```java
ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
Lock readLock = readWriteLock.readLock();
Lock writeLock = readWriteLock.writeLock();
```

针对读操作使用读锁，针对写操作使用写锁。

## 倒计数器与循环栅栏

倒计数器：CountDownLatch

循环栅栏：CyclicBarrier

这两个工具具体用法不好解释，我以一个场景考试举例，这个场景有如下特点：

- 参加考试的人员是一定的
- 使用一个线程表示一个参考人员
- 不限时，考试结束的唯一标志是所有人员交卷

使用倒计数器实现时：

- CountDownLatch定义多少人参加考试
- 每次有人交卷CountDownLatch就减少一，交卷后人员自由离开考场
- 当CountDownLatch减少到0时表示考试结束

使用循环栅栏实现时：

- CyclicBarrier定义多少人参加考试，宣布考试结束
- 每次有人交卷CyclicBarrier就减少一，但是交卷后人员不能离开考场
- 当CyclicBarrier减少到0时表示考试结束，所有人统一离场，宣布考试结束
- 当一场考试结束，下一场考试立即准备完毕

可以看出CyclicBarrier还是更强大一些的。

相较于CountDownLatch，CyclicBarrier可以定义完成事件，可以重复使用，控制线程的阻塞。

---

以上并发工具的使用示例都可以在[Joolkit](https://github.com/zhanghTK/Joolkit/tree/master/concurrent/src/test/java/tk/zhangh/java/concurrent/thread)中找到。
