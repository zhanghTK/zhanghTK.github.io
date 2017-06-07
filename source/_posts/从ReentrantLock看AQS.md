---
title: 从ReentrantLock看AQS
date: 2017-06-07 23:34:20
tags:
  - Java
categories:
  - Java
---

之前的文章有简单描述了JUC下的各种同步器。`ReentrantLock`的引入弥补了原生的`synchronized`关键字的不足。好些天不更新博客了，今天简单记录一下`ReentrantLock`的实现。

## 同步器的基本实现

JUC包下各种同步器主要依赖的都是`AbstractQueuedSynchronizer`（AQS）的实现，对于具体的实现大体可以分为两类：独占类型和共享类型。各种同步器并不直接继承成AQS，而是依赖于AQS的子类。

例如：`CountDownLatch`的内部类`java.util.concurrent.CountDownLatch.Sync`就是共享型的同步器，其继承了AQS。

本文记录同步器`ReentrantLock`的加锁、释放锁的具体实现。

## ReentrantLock

与`CountDownLatch`类似，`ReentrantLock`对同步器的实现也是依赖其继承了AQS的内部类`java.util.concurrent.locks.ReentrantLock.Sync`。根据公平性，又具体提供了`FairSync`和`NonfairSync`。

默认情况是`ReentrantLock`使用的是非公平的`NonfairSync`。

下面看看`ReentrantLock`的`NonfairSync`的加锁和释放锁，分析一下AQS对独占型同步器的支持。

### NonfairSync

#### lock

非公平锁加锁方法的入口：

```java
final void lock() {
  if (compareAndSetState(0, 1))
    setExclusiveOwnerThread(Thread.currentThread());
  else
    acquire(1);
}
```

首先是一个CAS操作，比较设置state变量的值。

state变量来自AQS，表示资源（锁）的状态，0表示未获取，1表示已获取一个。大于1表示获取（重入）的次数。

若资源未获取，修改资源状态成功后，会保存独占资源的线程（使用`setExclusiveOwnerThread`方法，该方法位于`AbstractOwnableSynchronizer`，AQS继承于它）。

若资源已经被获取，则调用`acquire`方法获取锁。

`acquire`是AQS在独占模式下获取资源的入口。可以看出在`ReentrantLock`加锁的主要逻辑都是依赖于AQS的，代码如下：

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

分别看一下`tryAcquire`,`acquireQueued`,`addWaiter`,`selfInterrupt`的具体实现：

1. 在ReentrantLock中`tryAcquire`具体的逻辑：

```java
protected final boolean tryAcquire(int acquires) {
  return nonfairTryAcquire(acquires);
}
```

`AQS`的`tryAcquire`是一个空方法，直接抛异常。`ReentrantLock`中对非公平锁的的实现则调用`nonfairTryAcquire`方法，逻辑如下：

```java
final boolean nonfairTryAcquire(int acquires) {
  // 获取当前要获取锁的线程
  final Thread current = Thread.currentThread();
  // 当前锁的状态
  int c = getState();
  // CAS操作，如果锁未获取修改锁状态，获取锁
  if (c == 0) {
    if (compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  // 如果锁已经被获取且当前线程是获取锁的线程，更新锁状态，重入锁
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  // 其他情况都失败
  return false;
}
```

2. `addWaiter`

```java
private Node addWaiter(Node mode) {
  // 根据线程实例和模式封装节点
  Node node = new Node(Thread.currentThread(), mode);
  // 尝试快速添加节点到队尾，成功则直接返回
  Node pred = tail;
  if (pred != null) {
    node.prev = pred;
    if (compareAndSetTail(pred, node)) {
      pred.next = node;
      return node;
    }
  }
  // 调用enq方法添加至队尾
  enq(node);
  return node;
}
```

```java
private Node enq(final Node node) {
  // 自旋操作
  for (;;) {
    Node t = tail;
    // 如果尾节点空（链表为空），创建一个不包含线程的头结点
    if (t == null) { // Must initialize
      if (compareAndSetHead(new Node()))
        tail = head;
    } else {
      // 添加至队尾
      node.prev = t;
      if (compareAndSetTail(t, node)) {
        t.next = node;
        return t;
      }
    }
  }
}
```

3. `acquireQueued`

```java
final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;  // 是否获取锁失败
  try {
    boolean interrupted = false;
    // 自旋操作
    for (;;) {
      final Node p = node.predecessor();  // 前驱节点
      // 如果前驱是头结点，并且获取资源成功
      // 因为enq方法中判断若链表为空，则创建一个没有线程的头结点。
      // 所以当前驱是头结点时，说明该节点有可能获取资源
      if (p == head && tryAcquire(arg)) {
        setHead(node);
        p.next = null; // help GC
        failed = false;
        return interrupted;
      }
      if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
        interrupted = true;
    }
  } finally {
    if (failed)
      cancelAcquire(node);
  }
}
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;
  if (ws == Node.SIGNAL)
    // 如果前驱节点状态是SIGNAL，返回
    return true;
  if (ws > 0) {
    // 如果前驱节点状态是CANCEL，删除此节都，直到前驱节点不是CANCEL
    do {
      node.prev = pred = pred.prev;
    } while (pred.waitStatus > 0);
    pred.next = node;
  } else {
    // 其他情况，使用CAS操作将前驱设置为SIGNAL
    compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}
```

关于节点状态的说明：

```java
// 代表线程已经被取消
static final int CANCELLED =  1;
// 代表后续节点需要唤醒
static final int SIGNAL    = -1;
// 代表线程在condition queue中，等待某一条件
static final int CONDITION = -2;
// 代表后续结点会传播唤醒的操作，共享模式下起作用
static final int PROPAGATE = -3;
// 新结点会处于这种状态
						    0;
```

```java
private final boolean parkAndCheckInterrupt() {
  // 暂停
  LockSupport.park(this);
  // 返回中断状况
  return Thread.interrupted();
}
```

至此完成锁获取的完整过程。

#### unlock

非公平锁释放锁方法的入口更简单，完全依赖AQS：

```java
public void unlock() {
  sync.release(1);
}
```

AQS中资源的释放实现：

```java
public final boolean release(int arg) {
  if (tryRelease(arg)) {
    // 释放资源成功
    Node h = head;
    // 队列后不为空，且队列头状态不是新建
    if (h != null && h.waitStatus != 0)
      // 释放队列头的后继节点
      unparkSuccessor(h);
    return true;
  }
  return false;
}
```

如同`tryAcquire`方法，AQS中`tryRelease`方法也是个空方法。`ReentrantLock`中的实现：

```java
protected final boolean tryRelease(int releases) {
  // 获取当前的state减一
  int c = getState() - releases;
  // 如果当前线程不是独占线程，抛出异常
  if (Thread.currentThread() != getExclusiveOwnerThread())
    throw new IllegalMonitorStateException();
  // 申请的锁是否都被释放
  boolean free = false;
  // 锁都被释放，重置独占线程
  if (c == 0) {
    free = true;
    setExclusiveOwnerThread(null);
  }
  // 更新state
  setState(c);
  return free;
}
```

释放头节点的后继节点：

```java
private void unparkSuccessor(Node node) {
  // 获取头结点的状态
  int ws = node.waitStatus;
  // 如果头结点状态小于0，更新为0
  if (ws < 0)
    compareAndSetWaitStatus(node, ws, 0);
  // 获取后继节点
  Node s = node.next;
  // 寻找头结点的下一个有效节点
  if (s == null || s.waitStatus > 0) {
    s = null;
    // 从尾部遍历，寻找第一个有效节点
    for (Node t = tail; t != null && t != node; t = t.prev)
      if (t.waitStatus <= 0)
        s = t;
  }
  // 释放锁
  if (s != null)
    LockSupport.unpark(s.thread);
}
```

以上就是释放资源的过程，相对于加锁还是比较简单的。

###  FairSync

与非公平的同步器不同，对于公平的同步器。在申请锁时条件更苛刻，要先判断当前节点前是否还有其他节点：

```java
final void lock() {
    acquire(1);
}

protected final boolean tryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {
    // 判断当前线程前是否还有其他线程
    if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
      setExclusiveOwnerThread(current);
      return true;
    }
  }
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0)
      throw new Error("Maximum lock count exceeded");
    setState(nextc);
    return true;
  }
  return false;
}
```

锁的释放则与非公平的同步器完全一致。

---

除了`lock`和`unlock`方法，`ReentrantLock`还提供了其他功能的锁申请/释放的方法，但都是依赖于AQS。

通过`ReentrantLock`大体可以对AQS后个印象：

1. 整个AQS使用双向链表作为底层数据结构；
2. 每个节点保存线程，以及状态信息；
3. 所有节点共同维护资源的使用情况；
4. 通过模板模式完成基本的同步器，具体的实现由子类完成（例如是否允许重入）；
5. 内部大量使用了CAS而非加锁确保线程安全；
