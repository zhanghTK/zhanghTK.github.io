---
title: Java线程基本操作
date: 2017-03-29 00:03:51
tags:
  - Java
categories:
  - Java
---

## 线程状态

Java中线程的定义与操作系统线程状态并不是一一对应的关系。

Java在`java.lang.Thread.State`中给出了关于线程状态的描述：

![线程状态.png](https://ooo.0o0.ooo/2017/03/24/58d507cd6f253.png)

## Thread定义的基本方法

- 新建

  - start：用于启动线程

  - run：普通方法,直接调用run方法将不会开启新的线程，依旧是在原有线程上顺序执行

- 终止

  - stop：不推荐使用，释放所有的锁，如果锁内的代码没有执行完毕，会出现不一致
  - 会抛出ThreadDeath Error

- 中断

  - 给线程一个标记，让线程自己响应中断，具体怎么响应甚至于是否响应由被中断线程自己决定

  - 与中断相关的方法包括：

    | 方法                                     | 说明                        |
    | :------------------------------------- | ------------------------- |
    | `public void interrupt()`              | 中断线程， 将中断状态设置为true        |
    | ` public boolean isInterrupted()`      | 测试线程是否已经中断。线程中断状态不受该方法的影响 |
    | ` public static boolean interrupted()` | 测试当前线程是否已经中断，并清除中断状态      |

  - 中断的处理

    - 中断与异常

      InterruptedException往往是阻塞的方法响应中断的方式，吞掉该异常将导致上层无法响应中断；

      具体处理方式可以上抛，或者重设中断状态，但不建议吞掉；

- sleep

  - 休眠线程，使线程进入TIMED_WAITING状态
  - 线程休眠期间不会释放锁
  - 休眠期间可以响应中断，会抛出InterruptedException


- 挂起与恢复

  - suspend/resume，不推荐使用

  - 缺陷：

    - 挂起操作不会释放锁，如果加锁放生在suspend之前，没有其他线程可以访问锁，直到resume，容易造成死锁
    - 如果resume先于suspend执行，线程无法结束会持续处于RUNNABLE状态，like this：

    ```java
    public class BadSuspend {
        private static final Object lock = new Object();

        public static class ChangeObjectThread extends Thread {
            public ChangeObjectThread(String name) {
                super.setName(name);
            }

            @Override
            public void run() {
                synchronized (lock) {
                    System.out.println("in " + getName());
                    Thread.currentThread().suspend();
                }
            }
        }

        public static void main(String[] args) throws InterruptedException {
            ChangeObjectThread thread1 = new ChangeObjectThread("thread1");
            thread1.start();
            // 确保thread1一定执行
            TimeUnit.MILLISECONDS.sleep(100);
            ChangeObjectThread thread2 = new ChangeObjectThread("thread2");
            thread2.start();
            thread1.resume();
            thread2.resume();
            thread1.join();
            thread2.join();
        }
    }
    ```

    运行代码后，程序始终没有结束，查看线程详细信息，发现thread1线程已不存在，thread2卡在suspend方法并且状态仍然是RUNNABLE的，like this：

    ![BadSuspend.jpg](https://ooo.0o0.ooo/2017/03/24/58d511f21c11c.jpg)

- Join/Yeild

  - yeild表示：别说我欺负你，我把机会让出来，我们一起抢
  - join表示：我就是欺负你，你等着，我先来
  - join的具体的实现参照后面的Object.wait方法介绍

  守护线程

  - 当只有守护线程时，JVM会自动退出
  - `thread.setDaemon(true);`

- 线程优先级：thread.setPriority(Thread.MAX_PRIORITY);

- 同步：synchronized

  - 加锁对象：对象锁
  - 加锁实例方法：加锁在方法所在对象实例，如果多个线程加在多个实例上并没有什么卵用
  - 加锁静态方法：加锁在方法所在Class对象上

## Object中的线程相关方法

Object.wait()/Object.notify()

- 先看JDK中对两个方法的描述：

  - wait()：

    Causes the current thread to wait until another thread invokes the java.lang.Object.notify() method or the java.lang.Object.notifyAll() method for this object.

  - notify()：

    Wakes up a single thread that is waiting on this object's monitor.

- 简言之，就是通过一个（monitor）对象，让线程停下来或是动起来

- 这两个方法在操作之前都一定要取得monitor对象的锁

- wait方法执行后线程会释放持有的锁，直到被他线程notify monitor时重新持有锁

- wait的典型使用场景：Thread.join

  join方法内部的关键实现：

  ```java
  public final synchronized void join(long millis) {
    if (millis == 0) {
      while (isAlive()) {
        wait(0);
      }
    } else {
      while (isAlive()) {
        long delay = millis - now;
        if (delay <= 0) {
          break;
        }
        wait(delay);
        now = System.currentTimeMillis() - base;
      }
    }
  }
  ```

- 两点需要注意的：

  - join最终的实现依赖：wait(delay);
  - 加锁位置：public final synchronized void join(long millis)，相当于synchronized(this)

  以一个例子说明：

  ```java
  public class JoinDemo {
      private volatile static long count = 0;

      public static void main(String[] args) throws InterruptedException {
          Thread thread = new Thread(() -> {
              while (++count < 1_000_000_000) {
              }
          });
          thread.start();
          thread.join();
          System.out.println(count);
      }
  }
  ```

  Main线程中执行thread.join，在join(int)方法中锁加在了thread对象上，内部的wait()是在thread对象上执行的。也就是说对于Main线程而言，thread线程是它的monitor。

  在join方法执行后Main线程处于WAITING状态，直到有人对它的monitor（thread 线程）执行notify操作，这一步是在thread线程执行完毕由JVM操作的。

  可以说thread既当爹又当妈，既要让当前线程停下来，又要自己跑起来。在thread执行完毕后重新对自己notify，让Main线程跑起来。
