---
title: 聊聊JDK动态代理
date: 2017-05-13 00:51:29
tags:
 - Java
categories: Java
---

提起代理多少有点老生常谈的意思了，无论是语言层面，模式层面，框架层面乃至系统架构，代理都算是比较高频的话题了。
之前的文章有提到 Spring AOP 的实现是基于 JDK 的动态代理和 CgLib 实现的。今天聊聊 JDK 中的动态代理实现。

## JDK 的描述

提起JDK动态代理，有两个类是怎么都绕不过去的：

`InvocationHandler` 和 `Proxy` 分别是实现 JDK 动态代理的门面。
`Proxy` 比较好理解，主要就是生成代理对象实例的，但 `InvocationHandler` 这个参数就有点意思了。
`InvocationHandler` 这个接口在 JDK 中的描述就是 invocation handler。

JDK 中对 `InvocationHandler` 的详细描述：
> InvocationHandler is the interface implemented by the invocation handler of a proxy instance.
> Each proxy instance has an associated invocation handler. 
> When a method is invoked on a proxy instance, the method invocation is encoded and dispatched to the invoke method of its invocation handler.

每一个代理都有一个被关联的invocation handler。当一个方法在代理对象上被调用，方法的调用被编码，由代理的 invocation handler 的 `invoke` 方法调用。

## 嵌套静态代理

先放结论：**实际上 JDK 的动态代理是由嵌套静态代理实现的。**

具体来说：我们根据业务抽象出 `Subject` 接口，并提供 `RealSubject` 作为实现，除此还需要提供一个 `InvocationHandler` 的实现 `SubjectInvocationHandler`。
把上面几个类组装一下，JDK 就可以根据以上信息默默的帮我们生成了代理类 `$Proxy`。

在代理生成前在程序中可以设置输出代理类的 class 文件：
`System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");`
先看看生成代理的源码，最终生成的代理类文件反编译后类似下面这样：

```java
public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m4;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello(String var1) throws  {
        try {
            super.h.invoke(this, m3, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayBye(String var1) throws  {
        try {
            super.h.invoke(this, m4, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m3 = Class.forName("tk.zhangh.pattern.structure.proxy.Subject").getMethod("sayHello", new Class[]{Class.forName("java.lang.String")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m4 = Class.forName("tk.zhangh.pattern.structure.proxy.Subject").getMethod("sayBye", new Class[]{Class.forName("java.lang.String")});
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

 `SubjectInvocationHandler` 是连接 `RealSubject` 和 `$Proxy` 的关键，扮演了既当爹又当妈的角色。

对于 `RealSubject`，`SubjectInvocationHandler` 是代理类；

对于 `$Proxy`, `SubjectInvocationHandler` 又是委托类。

当我们调用 `$Proxy` 的方法时，`$Proxy` 将调用转发给委托类 `SubjectInvocationHandler`, `SubjectInvocationHandler`作为代理类又将调用转发给 `RealSubject`。

`SubjectInvocationHandler` 不光是角色重要，还承担承上启下的任务，对`$Proxy`的调用，都是委托给了继承自`Proxy` 的`SubjectInvocationHandler`  的 `invoke` 方法处理。

而`SubjectInvocationHandler`的 `invoke` 方法完全由开发者编写，方法怎么调用，是否选择代理，都是在 `invoke` 方法实现的。所以 JDK 在对 invocation handler 的描述中用到了 dispatched，但是把调度逻辑方法 `invoke` 方法我实在不觉得是个优雅的做法。



有关动态代理实现详细的源码介绍可以参看[这里](http://www.cnblogs.com/MOBIN/p/5597215.html)，作者给出了比较详细的注释说明。
