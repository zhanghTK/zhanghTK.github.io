---
title: JDK中的Future模式
date: 2017-04-04 21:33:48
tags:
  - Java
  - 并发
categories: Java
---

## Future模式

Future模式：一个耗时的任务开始后立即返回客户端凭证，而不是要求客户端持续等待，客户端随时可以使用凭证获取任务结果

日常生活中类似的场景非常常见，比如你去维修电脑，把电脑交给售后拿好凭证你就可以走了，而不是守在售后等电脑。

Future模式的典型类图：

![Future模式.png](https://ooo.0o0.ooo/2017/03/30/58dcc547c217d.png)

- Task：任务接口
- FutureTask：客户端凭证，实际任务委托给ResultTask
- ResultTask：实际的任务处理
- Client：创建FutureTask包装ResultTask，开启新线程处理ResultData，返回FutureTask

如果任务没有执行完毕将持续阻塞，直到任务结束。具体实现可以使用wait/notifyAll方法以自己作为monitor阻塞自己。

## JDK的支持

JDK中提供了Future的支持，在Future模式之前先看看基本的支持。

先从线程池部分看起，线程池的继承结构如下：

- Executor接口中只定义了`void execute(Runnable command);`

- ExecutorService接口进一步扩展了Executor，添加了如`shutdown`，`isShutdown`等方法，需要注意的是ExecutorService添加了三个重载的`submit`方法：

  - ` <T> Future<T> submit(Callable<T> task);`
  - `<T> Future<T> submit(Runnable task, T result);`
  - `Future<?> submit(Runnable task);`

  自此，线程池与Future和Callable/Runnable产生了难分难解的缘分。

  继续ExecutorService实现之前先看看submit方法的返回值Future：

- Future接口内只定义了几个简单方法，获取任务状态及结果。

- AbstractExecutorService实现了ExecutorService，对三个submit方法的实现核心调用都是一致的：

  ```java
  RunnableFuture<T> ftask = newTaskFor(task, result);
  execute(ftask);
  return ftask;
  ```


关于线程池与Runnable与Callable的关系就到这里，下面是Runnable/Callable与RunnableFuture的关系了：

- RunnableFuture接口定义：

  `public interface RunnableFuture<V> extends Runnable, Future<V>`

  所以RunnableFuture有两个身份：

  1. Runnable，可以作为一个线程使用
  2. Future，可以获取运行结果和线程状态

  但是作为Runnable有什么好返回的呢？看一下具体的实现：FutureTask

- FutureTask有接收Runnable的构造：

  ```java
  public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
  }
  ```

- 再看`Executors.callable`：

  ```java
  public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
      throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
  }
  ```

- 继续看RunnableAdapter

  ```java
  static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
      this.task = task;
      this.result = result;
    }
    public T call() {
      task.run();
      return result;
    }
  }
  ```



我理解可以简单的概括为以下几点：

- Runnable只是单纯的启动线程
- Future只是单纯的获取线程状态
- RunnableFuture勾画了Runnable和Future统一的蓝图
- FutureTask最终完成了大一统
- 线程池既可以接收Runnable又可以接收Callable，但最终处理的都是Callable

## JDK的Future模式

![JDK的Future.png](https://ooo.0o0.ooo/2017/03/30/58dccdfa7fc8d.png)
