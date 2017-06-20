---
title: SpringBoot包文件执行分析
date: 2017-06-20 23:20:45
tags:
  - Java
  - Spring
categories: Spring
---
Spring Boot的一大特性是可以直接打包，并且这个Jar是可以直接启动的，不需要额外配置Servlet容器。

这一特性极大的简化了配置，运维的工作。之前碰到这个问题也没有深究，今天记一下，以备后看。

（最终本文未完成Servlet容器启动的说明，只记录了Jar包的启动）

## 项目结构

期初以为Spring Boot的启动入口就是应用中由`@SpringBootApplication`注解的`Main`方法，还在想到底是怎么把容器集成进去的。事先在网上查了一下，启动是从打包后Jar包中的 MANIFEST.MF 文件入手的。

首先看一下一个空的Spring Boot应用打包后的结构：

```
├── BOOT-INF
│   ├── classes
│   │   ├── application.properties
│   │   └── com
│   │       └── example
│   │           └── demo
│   │               └── DemoApplication.class
│   └── lib
│       ├── classmate-1.3.3.jar
│       ├── ...
├── META-INF
│   ├── MANIFEST.MF
│   └── maven
│       └── com.example
│           └── demo
│               ├── pom.properties
│               └── pom.xml
└── org
    └── springframework
        └── boot
            └── loader
                ├── JarLauncher.class
                ├── ...
```

整个结构从结构和命名就能大体猜到含义，与网上看到的结构有些差异，但整体大同小异。

回到刚才提及的 MANIFEST.MF，其内容如下：

```properties
Manifest-Version: 1.0
Implementation-Title: demo
Implementation-Version: 0.0.1-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: user
Implementation-Vendor-Id: com.example
Spring-Boot-Version: 1.5.4.RELEASE
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.example.demo.DemoApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_91
Implementation-URL: http://projects.spring.io/spring-boot/demo/
```

里面有构建的各项基本信息，其中的Main-Class指定了`JarLauncher`，`Start-Class`指定了应用中的入口。

看来`JarLauncher`才是项目的启动入口。除了Launcher外，Spring还提供了另一个重要的支持：Archive，先看看Archive。

## Archive

`Archive`是资源的抽象接口，定义了基本的资源获取抽象方法，可以用来表示Jar，文件目录等各种资源。

其子类`JarFileArchive`表示Jar包文件的抽象，内部包含一个`JarFile`对应一个Jar包，在创建`JarFile`实例时会解析Jar包的内部结构。

在资源解析过程中获取的URL可能存在多个`!/`作为资源分隔符分隔符，这是Spring在Jar协议基础上扩展出的，默认使用`org.springframework.boot.loader.jar.Handler`作为URL处理器。

资源解析较为繁琐，且与启动逻辑关系不紧密，这里不做多记录。

## Launcher

Launcher作为启动器的抽象，提供了多个具体实现：`ExecutableArchiveLauncher`,`JarLauncher`,`WarLauncher`以及`PropertiesLauncher` ，其中`ExecutableArchiveLauncher`是个抽象实现，作为`JarLauncher`和`WarLauncher`的父类。

`JarLauncher`是Jar包的启动类，下面详细看看启动过程实现：

```java
public static void main(String[] args) {
    new JarLauncher().launch(args);
}
```

直接调用父类`Launcher`的实现：

```java
protected void launch(String[] args) {
  try {
    // 设置上文提及的URL处理器
    JarFile.registerUrlProtocolHandler();
    // 获取classpath下的JarFileArchive，根据这些JarFileArchive创建类加载器
    ClassLoader classLoader = createClassLoader(getClassPathArchives());
    // 获得上文提及的start-class，调用重载launch方法
    launch(args, getMainClass(), classLoader);
  }
  catch (Exception ex) {
    ex.printStackTrace();
    System.exit(1);
  }
}

protected void launch(String[] args, String mainClass, ClassLoader classLoader) 
  						throws Exception {
    // 反射创建一个MainMethodRunner
	Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
    // 创建线程，启动
	Thread runnerThread = new Thread(runner);
	runnerThread.setContextClassLoader(classLoader);
	runnerThread.setName(Thread.currentThread().getName());
	runnerThread.start();
}

```

这里的ClassLoader是Spring自定义了类加载器`LaunchedURLClassLoader`，该类继承自`URLClassLoader`，加载逻辑：

```java
@Override
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
  synchronized (LaunchedURLClassLoader.LOCK_PROVIDER.getLock(this, name)) {
    Class<?> loadedClass = findLoadedClass(name);
    if (loadedClass == null) {
      Handler.setUseFastConnectionExceptions(true);
      try {
        loadedClass = doLoadClass(name);
      }
      finally {
        Handler.setUseFastConnectionExceptions(false);
      }
    }
    if (resolve) {
      resolveClass(loadedClass);
    }
    return loadedClass;
  }
}

private Class<?> doLoadClass(String name) throws ClassNotFoundException {
  // 尝试根类加载器加载
  try {
    if (this.rootClassLoader != null) {
      return this.rootClassLoader.loadClass(name);
    }
  }
  catch (Exception ex) {
  }
  
  // 尝试父类的findClass
  try {
    findPackage(name);
    Class<?> cls = findClass(name);
	return cls;
  }
  catch (Exception ex) {
  }

  // 尝试父类的加载（双亲委派）
  return super.loadClass(name, false);
}

// 设置根加载器为Extension ClassLoader
public LaunchedURLClassLoader(URL[] urls, ClassLoader parent) {
  super(urls, parent);
  this.rootClassLoader = findRootClassLoader(parent);
}

private ClassLoader findRootClassLoader(ClassLoader classLoader) {
  while (classLoader != null) {
    if (classLoader.getParent() == null) {
      return classLoader;
    }
    classLoader = classLoader.getParent();
  }
  return null;
}
```

简而言之破坏了双亲委派，如果没有加载过使用`doLoadClass`方法加载，内部加载逻辑：

1. 尝试根类加载器加载(Extension ClassLoader)
2. 尝试父类的findClass
3. 尝试父类的加载（双亲委派）

最后详细看一下`MainMethodRunner`的`run`方法，启动的其它逻辑都这这里了：

```java
@Override
public void run() {
  try {
    // 获取应用入口
    Class<?> mainClass = Thread.currentThread().getContextClassLoader()
      .loadClass(this.mainClassName);
    Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
    if (mainMethod == null) {
        throw new IllegalStateException(this.mainClassName
						+ " does not have a main method");
    }
    // 调用main方法
    mainMethod.invoke(null, new Object[] { this.args });
  } catch (Exception ex) {
    ex.printStackTrace();
	System.exit(1);
  }
}
```

总结一下：

1. 项目启动从`JarLauncher`开始
2. 设置Url处理器，加载需要的各种资源
3. 相关资源的加载使用了自定义的类加载器
4. 开启新线程调用应用入口

整个实现过程也没有用到什么特别的东西，主要还是反射，类加载器，线程的东西，额外还扩展了Jar协议。

但是并没有看到关于Servlet容器的内容，看来容器的启动和Jar的执行是分开的。
