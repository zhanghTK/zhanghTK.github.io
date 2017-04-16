---
title: JDK并发包——线程池
date: 2017-04-16 19:05:56
tags:
  - Java
  - 并发
categories: Java
---

## JDK的支持

![Executor.png](https://ooo.0o0.ooo/2017/04/10/58eb4e010f8ef.png)

以上为JDK线程池的核心类,

日常线程池的使用可以以ExecutorService为通用的接口，由Executors生产特定的线程池实现

Executors提供各种工具方法的支持和基本线程池实现包括：

- newFixedThreadPool
- newSingleThreadExecutor
- newCachedThreadPool
- newSingleThreadScheduledExecutor
- newScheduledThreadPool

ScheduledExecutorService与其他几个线程池不同，提供了三个特殊方法：

- schedule：在给定时间调度一次任务
- scheduleAtFixedRate：以任务开始时间为起点，按给定频率调度任务
- scheduleWithFixedDelay：以任务结束时间为起点，按给定频率调度任务

注意：

1. 如果周期太短，那么任务会在上个任务结束后立即调用
2. 如果任务抛出异常，那么后续所有执行都会被中断

## 线程池实现

ThreadPoolExecutor最重要的构造方法：

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory
                              RejectedExecutionHandler handler);
```

Executors提供的不同线程实现就是根据不同参数定制的，有两个参数需要注意：

1. workQueue：被提交但未执行的任务队列

   可以直接使用的几种BlockingQueue：

   - SynchronousQueue：直接提交队列
   - ArrayBlockingQueue：有界队列
   - LinkedBlockingQueue：无界队列
   - PriorityBlockingQueue：优先级队列

   ![ThreadPoolExecutor任务调度逻辑.png](https://ooo.0o0.ooo/2017/04/10/58eb6684c2895.png)

2. handler：拒绝策略，包括：

   - AbortPolicy
   - CallerRunsPolicy
   - DiscardOledestPolicy
   - DiscardPolicy
   - 实现RejectedExecutionHandler接口自定义

## 线程池基本使用

### 切面扩展

- beforeExecute()
- afterExecute()
- terminated()

### 线程池与异常

在线程池中执行线程有两种方法：

- submit
- execute

两者除了是否有返回值之外，在异常的处理方式上也存在区别。

submit是ExecutorService中引入的方法，在AbstractorService中各个重载的submit方法最终都会以`return execute(new FutureTask)`的形式执行。

所以submit线程最终是由线程池调用FutureTask的run方法执行，execute的run方法是由线程调用FutureTask的run方法执行。

看一下FutureTask的源码：

```java
public void run() {
  ...
  try {
    Callable<V> c = callable;
    if (c != null && state == NEW) {
      V result;
      boolean ran;
      try {
        result = c.call();
        ran = true;
      } catch (Throwable ex) {
        result = null;
        ran = false;
        setException(ex);
      }
      if (ran)
        set(result);
    }
  } finally {
  ...
  }
}

protected void setException(Throwable t) {
  if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    outcome = t;
    UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
    finishCompletion();
    }
}

public V get() throws InterruptedException, ExecutionException {
  int s = state;
  if (s <= COMPLETING)
    s = awaitDone(false, 0L);
  return report(s);
}

private V report(int s) throws ExecutionException {
  Object x = outcome;
  if (s == NORMAL)
    return (V)x;
  if (s >= CANCELLED)
    throw new CancellationException();
  throw new ExecutionException((Throwable)x);
}
```

除非你手动get，否则你是得不到任何异常的。使用execute则没有这个问题。

可以重新实现线程池以获取更详细的异常堆栈信息。

### 线程数量

公式：Nthread = Ncpu * Ucpu * （1+ W/C），各字段含义：

Nthreads：线程数量

Ncpu：CPU的数量，Runtime.getRuntime().availableProcessors()

Ucpu：CPU使用率，范围在[0,1]

W/C：等待时间与计算时间的比率

## Fork/Join线程池

Fork/Join是分治思想的线程池框架。

核心接口和实现包括：

1. ForkJoinPool：专门为ForkJoin框架提供的线程池
2. ForkJoinTask：抽象的计算任务
   1. RecursiveTask：有返回值得具体任务
   2. RecursiveAction：无返回值的具体任务

使用ForkJoin线程池的优点：

1. 避免大量的开启，回收线程，线程的开启和回收都依赖线程池
2. 某线程任务处理后可以从其他线程获取任务处理（双端队列，工作密取）
