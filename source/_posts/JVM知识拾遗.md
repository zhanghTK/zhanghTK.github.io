---
title: JVM知识拾遗
date: 2017-12-08 21:36:30
tags:
  - Java
  - JVM
categories: JVM
---

## 运行时数据区域

谈论到运行时数据区可能有非常多词在脑海中闪现：堆、栈、新生代、老年代、永久区、方法区等等。上面这些概念确实都位于运行时数据区，但从 JVM 功能实现的角度没有这么多概念，例如老年代，新生代都是从垃圾收集的角度去分析。单纯从 JVM 功能实现角度可以用下图概括：

![运行时数据区](https://camo.githubusercontent.com/8799029281fbeed32ea53e95df7c82042a3dff21/68747470733a2f2f6f6f6f2e306f302e6f6f6f2f323031372f30372f30322f353935383832373332333737312e706e67)

### 线程私有区域

- 程序计数器

  当先线程所执行的字节码的行号执行器

- Java 虚拟机栈

  与线程一致的声明周期。由若干个栈帧组成，每个栈帧对应一个方法。栈帧存储局部变量表、操作数栈等信息

### 线程共享区域

- Java堆

  Java 堆主要用于存放对象实例，所有线程共享的内存区域统称为 Java 堆，但其内部又划分为若干个逻辑区域。从功能的角度上看 Java 堆存放对象实例，可以分为新生代、老年代。


- 方法区

  Java 堆的一个逻辑区域，但功能上与 Java 堆有明显的区别。主要是存储被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等数据。内部重要区域：运行时常量池。

  在方法区这部分有几个混淆的概念：方法区，永久代，元空间：

  - 方法区：JVM 规范中定义的区域
  - 永久代：HotSpot 在 JDK1.8 之前对方法区的实现，使用 Java 堆内存
  - 元空间；HotSpot 在 JDK1.8 开始对方法区的实现，该区域使用本地内存

《深入理解 Java 虚拟机》一书在第 2 章从内存溢出的角度详细的说明了运行时数据区域（甚至包括了非虚拟机运行时数据区域），个人感觉区域之间层次描述稍弱，内容也较多，从 JVM 运行模型的角度上面的概括更容易理解。

## 内存分配与回收

单纯从 JVM 功能实现的角度考虑，对象内存的分配和回收就是对堆空间的操作。但为了高效的自动回收内存，一般把 Java 堆分为新生代和老年代，二者在对象内存分配和回收策略上有所区别。区别的根本原因是为了内存回收，所以先从内存回收策略看起——垃圾收集算法。

### 垃圾收集算法

- 标记-清除
  - 最基本的垃圾收集算法
  - 效率不高；内存碎片
- 复制算法
  - 适合“朝生夕死”的对象，新生代收集算法
  - 由三部分组成：两个小块 Survivor 和一个大块 Eden，HotSpot 默认比例8:1:1，每次使用一个 Survivor 和一个 Eden
  - 当 Survivor 空间不足时依赖其他内存进行担保
- 标记-整理
  - 适合存活率高对象
  - 标记-清除-整理

### 内存分配

内存分配没有绝对的准则，采用不同的垃圾收集器，JVM 实现，JVM 参数可能得到不同的分配策略。但有几条普遍策略：

- 对象优先在  Eden 分配
- 大对象直接进入老年代
- 长期存活的对象将进入老年代
- 动态对象年龄判定：如果在 Survivor 中间中相同年龄所有对象大小的综合大于 Survovor 空间一半，年龄大于等于该年龄的对象直接晋升老年代
- 空间分配担保

### 垃圾收集器

先放一张垃圾收集器的组合

![GC.png](https://ooo.0o0.ooo/2017/09/19/59c0ee65ee2a4.png)

#### 新生代收集器

##### Serial

![Serial.jpg](https://i.loli.net/2017/09/19/59c1060eed6f0.jpg)

单线程垃圾收集器，进行垃圾收集的时候需要暂停其他的线程

##### ParNew

![PaeNew.jpg](https://i.loli.net/2017/09/19/59c10658676f9.jpg)

Serial收集器的多线程版本，许多运行在Server模式下的虚拟机中首选的新生代收集器，可以与CMS收集器配合工作

##### Parallel Scavenge

并行的多线程收集器，更关注可控制的吞吐量。吞吐量越大，垃圾收集的时间越短。目前没有使用过。

#### 老年代收集器

##### Serial Old

Serial收集器的老年代版本，也是一个单线程收集器，采用“标记-整理算法”进行回收。其运行过程与Serial收集器一样。

用途：

- 与Parallel Scavenge收集器搭配使用
- 作为CMS收集器的后备预案，在并发收集发生 Concurrent Mode Failure 时使用

##### Parallel Old

![Parallel Old.jpg](https://i.loli.net/2017/09/19/59c106f16c0e9.jpg)

Parallel Scavenge收集器的老年代版本，使用多线程和“标记-整理”算法进行垃圾回收。与 Parallel Scavenge 收集器配合使用，“吞吐量优先”收集器是这个组合的特点，在注重吞吐量和CPU资源敏感的场合，都可以使用这个组合。

##### CMS

![1505822517(1).jpg](https://i.loli.net/2017/09/19/59c1075791807.jpg)

为获取最短回收停顿时间而生的老年代垃圾收集器（标记-清除算法）。整个执行过程分为 4 个步骤：

- 初始标记：仅标记 GC Roots 直接可达的对象，需要 STW
- 并发标记：GC Roots Tracing，并行
- 重新标记：修正上一步执行中变动的标记记录
- 并发清除：并行

缺点：对 CPU 资源敏感，无法有效处理浮动垃圾，无法有效处理内存碎片

目前我在项目中使用最多的垃圾收集器，从 GC 日志来看，STW 时间确实少，以最近一次本地 GC 日志说明：85% 的 GC 时间持续一秒内，但最大的 STW 时间只有 160 ms，平均 STW 时间 135 ms。

##### G1 收集器

目前这个收集器完全没有使用过。从描述上看是神一般的存在。大致思想是将Java堆划分为多个大小相等的Region（独立区域），新生代与老年代都是一部分Region的集合，G1的收集范围则是这一个个Region（化整为零）。

![1505823191(1).jpg](https://i.loli.net/2017/09/19/59c109faa5c94.jpg)

---

以上收集器虽然多，但还是有规律可以总结的：

- Serial：是最原始的新生代收集器
- Parnew：多线程版本的 Serial 新生代收集器
- Parallel Scavenge：吞吐量优先新生代收集器
- Serial Old：最原始的老年代收集器，万能备胎
- Parallel Scavenge：吞吐量优先老年代收集器
- CMS：多线程版本的老年代收集器

## 备忘录

### 虚拟机参数小结

| 参数                         | 描述                        |
| -------------------------- | ------------------------- |
| -Xms                       | 初始堆大小                     |
| -Xmx                       | 最大堆大小                     |
| -Xmn                       | 新生代大小                     |
| -Xss                       | 每个线程的堆栈大小                 |
| -XX:NewRatio               | 新生代和老年代的比例                |
| -XX:SurvivorRatio          | 新生代 Eden 区和 Survivor区域的比例 |
| -XX:PermSize               | 永久代的初始大小                  |
| -XX:MaxPermSize            | 永久代的最大值                   |
| –XX:MetaspaceSize          | 元空间的初始大小                  |
| -XX:MaxMetaspaceSize       | 元空间的最大值                   |
| -XX:MaxTenuringThreshold   | 超过设置年龄的新生代直接晋升老年代         |
| -XX:PretenureSizeThreshold | 超过设置大小的新生代直接晋升老年代         |
| -XX:+PrintGC               | 每次 GC 时打印相关信息             |
| -XX:+PrintGC Details       | 每次 GC 时打印详细信息             |
| -XX:+PrintGCTimeStamps     | 打印每次 GC 的时间戳              |
| -Xloggc                    | 是将每次GC事件的相关情况记录到文件中       |

### 垃圾收集器参数总结

| 参数                                 | 描述                                       |
| ---------------------------------- | ---------------------------------------- |
| -XX:+UseSerialGC                   | 使用 Serial + Serial Old 收集器组合             |
| -XX:+UseParNewGC                   | 使用 ParNew + Serial Old 收集器组合             |
| -XX:+UseConcMarkSweepGC            | 使用 ParNew + CMS + Serial Old 收集器组合       |
| -XX:+UseParallelGC                 | 使用 Parallel Scavenge + Serial Old 收集器组合  |
| -XX:+UseParallelOldGC              | 使用 Parallel Scavenge + Parallel Old 收集器组合 |
| -XX:ParallelGCThreads              | 设置并行 GC 时进行内存回收的线程数                      |
| -XX:GCTimeRatio                    | Parallel Scavenge 中 GC 时间占比，默认 99，即允许 1% 的 GC 时间。 |
| -XX:MaxGCPauseMillis               | 设置 GC 的最大停顿时间，只对 Parallel Scavenge 有效    |
| -XX:CMSInitiatingOccupancyFraction | 设置 CMS 收集器在老年代空间被使用多少后触发垃圾收集             |
| -XX:+UseCMSCompactAtFullCollection | 设置 CMS 收集器在完成垃圾收集后是否要进行一次内存碎片整理          |
| -XX:+CMSFullGCBeforeCompaction     | 设置 CMS 收集器在完成若干次垃圾收集后是进行一次内存碎片整理         |

### 启动参数的选择

启动参数大体指定三部分：设置收集器相关配置、设置内存分配相关配置，其他。

对于收集器，目前我我基本上都使用 ParNew（新生代）+ CMS（老年代）+ Serial Old（老年代备用）组合。可能需要指定的参数还有：-XX:ParallelGCThreads，-XX:CMSInitiatingOccupancyFraction，-XX:+UseCMSCompactAtFullCollection，-XX:+CMSFullGCBeforeCompaction。但目前实际上我都没有特殊指定（设置过线程数，但发现效果并不理想）。

对于内存分配，这个就要结合项目和环境的具体信息配置了。

其他部分就包括：对 GC 过程、类加载卸载等信息的输出，异常 Dump 等信息。

完成以上配置后，还需要根据系统运行状况，GC 日志的情况进一步调整参数。对 GC 日志的分析可以使用 [gceasy](http://gceasy.io/index.jsp) 分析，系统运行状况可以参考后面的文章《JVM 故障发现和排查》。

---

参考：

- 《深入理解 Java 虚拟机》
- [Java永久代去哪儿了](http://www.infoq.com/cn/articles/Java-PERMGEN-Removed)
- [Java8内存模型—永久代(PermGen)和元空间(Metaspace)](http://www.cnblogs.com/paddix/p/5309550.html)
