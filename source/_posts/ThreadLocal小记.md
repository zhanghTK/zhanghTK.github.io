---
title: ThreadLocal小记
date: 2016-09-11 19:33:39
tags: Java
categories: Java
---

# ThreadLocal

最近《Java并发编程实战》第三章谈及了对象共享的问题。对共享的可变数据最简单粗暴的做法当然是同步，但是同步的缺点也很明显，代码复杂可维护性降低。针对这个问题，书上谈及到了通过线程封闭避免同步，其中的ThreadLocal类就是帮助维持线程封闭性的。

之前对ThreadLocal的认识非常简单，就是把一个变量绑定到线程上。参照网上的例子自己也实现了类似功能的例子ThreadLocalVariable(https://github.com/zhanghTK/HelloJava/blob/master/src/main/java/tk/zhangh/java/thread/ThreadLocalVariable.java)，但总的来说没什么体会。今天参照别人的文章看了一下ThreadLocal的实现，发现还是有蛮多注意点的。

看JDK之前想当然的以为ThreadLocal应该就是简单对`Map<Thread, Object>`做一个封装，然而实际并没有这么简单。参照了网上一些文章的说法，早期的ThreadLocal确实是这样实现的。但是这样实现存在一些问题：

1. 线程安全问题，如果使用线程安全的Map实现那么就会带来性能问题，当有大量的线程使用ThreadLocal，伴随着线程生命周期ThreadLocal也需要频繁向底层的Map添加删除数据。
2. 内存回收问题，用Thread当key，除非手动调用remove，否则即使线程退出了会导致：1)该Thread对象无法回收；2)该线程在所有ThreadLocal中对应的value也无法回收。

ThreadLocal实际给出了不同的实现方式。首先绑定到线程的变量没有维护在ThreadLocal内，而是维护在各个Thread类实例内——在Thread类内使用了ThreadLocal的静态内部类`ThreadLocalMap`实例去维护需要绑定到线程的变量。这样原本需要维护在ThreadLocal内的数据现在就分散到了各个线程内去维护。



在Thread中ThreadLocalMap的声明长这样：

```java
// 真的就只是声明了一下，什么都没干    
ThreadLocal.ThreadLocalMap threadLocals = null;
```



ThreadLocal一共只有五个非私有的方法，首先是两个并没有什么卵用的方法：

```java
// 构造，什么都没干
public ThreadLocal() {}

// 设置ThreadLocal的初始值，protected很明显是希望子类重写
protected T initialValue() {return null}
```

看看其余三个方法的实现（JDK8）

```java
    public T get() {
        Thread t = Thread.currentThread();
        // 从线程里获取ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 根据ThreadLocal实例获取Entity
            // 一会看ThreadLocal的实现
            // 暂时可以看做类似Map<ThreadLocal,Object>
            // 注意key类型是ThreadLocal，不是Thread
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 如果获得的map为null或者从map通过key获取的value为空时获取一个初始值
        // setInitialValue方法里调用了initialValue方法
        // 宝宝不管，反正宝宝想要有值，宝宝不想为null
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            // 没什么说的set进去
            map.set(this, value);
        else
            // 当map不存在时，使用初始值创建一个
            createMap(t, value);
    }

     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             // 没什么说的，remove掉
             m.remove(this);
     }
    
    ThreadLocalMap getMap(Thread t) {
        // 你给我一个线程，我换你一个map
        return t.threadLocals;
    }
```
ThreadLocal所有的操作都是围绕着内部类ThreadLocalMap的，ThreadLocal只是让ThreadLocalMap更加容易访问。咦，有点耳熟，没错外观模式。

下面看一下ThreadLocal的静态内部类ThreadLocalMap，JDK文档对它的描述主要集中在一下几点：

1. 是什么：定制的hash map用于维护本地线程变量
2. 可见性：ThreadLocal之外没有任何方法可访问，Thread类使用它定义了私有属性
3. 特殊性：为了管理大对象、长生命周期对象，使用WeakReference包装ThreadLocal对象作为key

前面两点没什么好说的，主要是第三点：key使用了弱引用类型管理（关于弱引用可以先参考https://github.com/zhanghTK/HelloJava/blob/master/src/main/java/tk/zhangh/java/jvm/reference/ReferenceTest.java，后面我会单独写文章记录Java引用的使用，先挖个坑）。

![image](http://img.blog.csdn.net/20160121000731607)

图片来自互联网，实线表示强引用，虚线表示弱引用。

简而言之，一个对象在没有强引用引用，只有弱引用引用时，当GC发生这个对象就会被标记回收。将ThreadLocal对象设置成弱引用作为key的好处是显而易见：当ThreadLocal没有任何强引用引用时，只有ThreadLocalMap的Entry对它存在弱引用，这样GC的时候这个ThreadLocal对象就可以被回收了。但是这又带来了一个问题：Entry的key可能被回收了，但留下了一个并没有什么卵用的value。只要线程生命周期不结束，那么这个value对象始终保持了一个强引用链条：

Thread Ref -> Thread -> ThreadLocalMap -> Entry -> value

当内存有大量驻留的线程时，因为强引用存在，GC始终无法回收，就导致了内存泄漏。

真对这个问题ThreadLocalMap在实现的时候也采取了一些防护措施，比如ThreadLocalMap的get（set，remove方法也类似，限于篇幅不展开了）：

```java
    private Entry getEntry(ThreadLocal<?> key) {
        // hash函数获取索引位置
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            // 命中了
            return e;
        else
            // miss了
            return getEntryAfterMiss(key, i, e);
    }

    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
		
        // 遍历table
        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                // 找到了
                return e;
            if (k == null)
                // 发现key为空（也就是上面描述的内存泄漏的情况），做删除
                expungeStaleEntry(i);
            else
                // 找下一个Entty位置
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }
```

在get的过程中凡是碰到了key为null的Entry，这个Entry就会被擦除，从而避免内存泄漏。类似的思路在set，remove方法中都有实现。针对内存泄漏问题ThreadLocal实际是需要手动触发函数删除key为null的Entry，所以当不要再需要一个变量当定到线程时手动的remove还是很有必要的。



最后，在网上看到对ThreadLocal有多种说法：ThreadLocal为解决多线程程序的并发问题提供了一种新的思路；ThreadLocal的目的是为了解决多线程访问资源时的共享问题。

关于第一种说法我觉得是部分正确的，ThreadLocal将需要共享的对象封闭在了线程内确实解决了并发的一部分问题，但并不是万能的。比如重写initialValue方法时返回的是一个全局共享的对象，那实际上ThreadLocal只是把这个全局共享的对象又封装到了Thread对象里，ThreadLocal本身并没有做类似深拷贝的操作，因此这个变量依然是线程不安全的（逸出）。

关于第二种说法，我觉得问题就比较大了。ThreadLocal根本不是为了线程间共享，实际是为了将状态封闭在线程内以确保线程安全。这样做确实带来了访问共享状态的便利，但这个状态的共享是在单个线程内的，而不是线程之间的。比如在action中将session对象绑定在线程内，在service、dao里都可以方便的共享，但所有的共享都是在单个的线程内部，而不是在多个的线程之间共享。



参考资料：

《Java并发编程实战》

JDK8帮助手册

http://qifuguang.me/2015/09/02/[Java%E5%B9%B6%E5%8F%91%E5%8C%85%E5%AD%A6%E4%B9%A0%E4%B8%83]%E8%A7%A3%E5%AF%86ThreadLocal/

http://my.oschina.net/xianggao/blog/392440#navbar-header
