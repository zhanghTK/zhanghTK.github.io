---
title: JVM故障发现排除
date: 2017-12-08 21:20:40
tags:
  - Java
  - JVM
categories: JVM
---

最近做了服务器迁移之后，系统运行过程中出现了几次发现不稳定的情况。这次的经历又回想起之前几次碰到类似的问题，类似的问题往往需要能快速排查定位、处理。但相关类似问题又不是经常能够碰到，每次出现问题都是手忙脚乱的查资料，今天根据《深入理解 Java 虚拟机》和自己简单的经验做一下总结，方便日后使用。

## 工具

目前可用的 JVM 监控工具还是蛮多的，这里只列出实际操作中我使用的几个。其中有命令行工具也有第可视化工具，从使用的便捷性上讲，可视化的工具无疑是更好的。但生产环境处于安全，性能的考虑往往不开放远程连接，这时还是得用命令行工具处理。

### 命令行工具

- jps：JVM 进程状况工具，主要作用是查看 LVMID，-v 参数可以输出 JMV 启动参数

- jstat：JVM 统计信息监控工具，主要是查看 GC 相关信息：

  - -gc：监视 Java 堆状况
  - -gccapacity：监视 Java 堆状况，最大，最小空间
  - -gcutil：监视 Java 堆状况，已使用百分比

  输出列含义见文末

- jinfo：Java 配置信息工具，查看设置 JVM 启动参数

  - 查看：-flag < name >：输出指定名称参数值，作为 jps -v 补充
  - 设置：
    - -flag [+|-] < name >：设置指定 JVM 参数的布尔值
    - -flag < name > = < value >：设置指定 JVM 参数的值

- jmap：Java 内存映像工具，主要参数：

  - -dump：生成快照，例如：-dump:format=b,file=< filename.bin > < pid >
  - -heap：显示 Java 堆详细信息
  - -histo：像是堆中对象统计信息
  - -permstat：以 ClassLoader 为统计口径显示永久代内存状况

- jstack：Java 堆栈跟踪工具，用于生成 JVM 当前线程快照，-l 参数显示关于加锁信息，-F 参数强制 dump

### 可视化工具

- VisualVM：多合一故障处理工具，这个几乎涵盖了我用到上面命令行的所有功能
- MemoryAnalyzerTool：用于分析 dump 堆文件，对比 VisualVM 功能单一，但是提供了报表功能可以协助分析问题

## 问题排查一般思路

目前我在实际开发过程中碰到的 JVM 问题主要可以分类两类：内存溢出和系统运行缓慢

### 内存溢出

这种场景是影响最恶劣，但也是最容易排查的。通常的错误就能告诉说明溢出区域：

- outOfMemoryError ：年老代内存不足
- outOfMemoryError:PermGen Space：永久代内存不足
- outOfMemoryError:GC overhead limit exceed：垃圾回收时间占用系统运行时间的98%或以上

生产环境碰到这类情况为了确保系统可用，可以先使用 jstat 查看 Java 堆的空间使用情况确定到底是哪部分溢出，其最大可用空间是多少，然后直接扩大该区域空间即可。重启时建议加上 -XX:+HeapDumpOnOutMemoryError 作为启动参数，在下次溢出可以获得快照文件。

上面的做法作为临时解决方案可以解决一般性的问题，但没有系统的快照无法深入分析问题产生的原因，如果想分析问题根源需要在重启前使用 jmap 生成快照。获得快照文件后可以在本地使用可视化工具分析。使用 MemoryAnalyzer 打开快照文件就能获得一个分析报表，里面列出了可能出现泄露的地方。目前我碰到的内存溢出问题一般从这个分析报表里面就可以确定了，如果不能确定就需要根据加载的类，类的实例，引用关系进一步分析了。

我目前碰到的都是 Java 堆的溢出，以上的思路基本可以解决。但除此还有其他的内存溢出需要注意：

- Direct Memory
- 线程堆栈：StackOverflowError，OutOfMemoryError：unable to create new native thread
- Socket 缓冲区：IOException：Too many open files

### 系统运行缓慢

系统运行缓慢可能出现的原因就比较多了，通常就是找到导致系统缓慢的具体代码段，然后修复。一般化的解决思路是从系统到应用，从应用到线程。

具体来说：首先使用系统监控工具（例如 top，vmstat）查看当前系统运行状况，确认哪个应用的资源占用过大，是否是 Java 应用的问题；其次使用 jps 获得具体应用的 LVMID，根据 LVMID 查看应用的具体运行状况，如线程情况，系统信息等。还可以根据系统工具 pidstat 进一步查看线程的运行信息来辅助确定问题。以上基本就可以确定问题。

### 本地应用

如果是本地应用或者可以远程访问的应用排查起来就更方便了，直接使用 VisualVM 连接上去，从系统到线程的一切信息都了如指掌，还可以直接运行 GC，dump 快照等。

---

- 关于工具使用的一些参考：

  [JVM性能调优监控工具jps、jstack、jmap、jhat、jstat、hprof使用详解](https://my.oschina.net/feichexia/blog/196575)

  [MAT - Memory Analyzer Tool 使用进阶](http://www.lightskystreet.com/2015/09/01/mat_usage/)

- jstat 输出列含义：

  S0C：年轻代中第一个survivor的容量 (字节) 

  S1C：年轻代中第二个survivor的容量 (字节) 

  S0U：年轻代中第一个survivor目前已使用空间 (字节) 

  S1U：年轻代中第二个survivor目前已使用空间 (字节) 

  EC：年轻代中Eden的容量 (字节) 

  EU：年轻代中Eden目前已使用空间 (字节) 

  OC：Old代的容量 (字节) 

  OU：Old代目前已使用空间 (字节) 

  PC：Perm(持久代)的容量 (字节) 

  PU：Perm(持久代)目前已使用空间 (字节) 

  YGC：从应用程序启动到采样时年轻代中gc次数 

  YGCT：从应用程序启动到采样时年轻代中gc所用时间(s) 

  FGC：从应用程序启动到采样时old代(全gc)gc次数 

  FGCT：从应用程序启动到采样时old代(全gc)gc所用时间(s) 

  GCT：从应用程序启动到采样时gc用的总时间(s) 

  NGCMN：年轻代中初始化(最小)的大小 (字节) 

  NGCMX：年轻代的最大容量 (字节) 

  NGC：年轻代中当前的容量 (字节) 

  OGCMN：old代中初始化(最小)的大小 (字节) 

  OGCMX：old代的最大容量 (字节) 

  OGC：old代当前新生成的容量 (字节) 

  PGCMN：perm代中初始化(最小)的大小 (字节) 

  PGCMX：perm代的最大容量 (字节) 

  PGC：perm代当前新生成的容量 (字节) 

  S0：年轻代中第一个survivor已使用的占当前容量百分比 

  S1：年轻代中第二个survivor已使用的占当前容量百分比 

  E：年轻代中Eden已使用的占当前容量百分比 

  O：old代已使用的占当前容量百分比 

  P：perm代已使用的占当前容量百分比 

  S0CMX：年轻代中第一个survivor的最大容量 (字节) 

  S1CMX ：年轻代中第二个survivor的最大容量 (字节) 

  ECMX：年轻代中Eden的最大容量 (字节) 

  DSS：当前需要survivor的容量 (字节)（Eden区已满） 

  TT： 持有次数限制 

  MTT ： 最大持有次数限制 
  
  
  ---
  20171208 补充
  
  目前系统接入了 APM（Application Performance Management） 对整个系统的运行进行监控。监控内容包括但不限 JVM 相关内容，非常值得参考。
  [APM 传送问](https://github.com/naver/pinpoint)


