---
title: SpringBoot启动行为分析——监听器篇
date: 2017-06-30 01:47:21
tags:
  - Java
  - Spring
categories: Spring
---

启动的准备工作做完就真的可以 run 了（话说我不是找 Servlet 容器的启动的么），看看`SpringApplication`的实例`run`方法又做了哪些事情。

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}
```

方法完整的逻辑还是有点复杂的，稍微拆分一下，方便分析。

以try-catch代码块分隔，catch的异常处理暂且不表，先看try前面的代码，其核心功能是启动了所有的监听事件。

## 监听器启动

```java
        // 获取SpringApplicationRunListener集合
		SpringApplicationRunListeners listeners = getRunListeners(args);
        // 依此调用SpringApplicationRunListener的starting方法
		listeners.starting();
```

监听器启动的代码其实就上面这两句。

首先是`getRunListeners`，还是之前实例化初始化器和监听器的方法实现，这次实例化所有`SpringApplicationRunListener`然后和一个`log`实例一起封装到`SpringApplicationRunListeners`实例。

```java
	private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
				SpringApplicationRunListener.class, types, this, args));
	}
```

获取所有`SpringApplicationRunListener`实例后，立即调用了`starting`方法。所以先看看`SpringApplicationRunListener`到底是个什么鬼。

## 监听器实现

`SpringApplicationRunListener`是个接口，定义了五个方法：

1. `starting`：run方法首次调用
2. `environmentPrepared`：ApplicationContext创建之前并且环境信息准备好的时候调用
3. `contextPrepared`：ApplicationContext创建好并且在source加载之前调用一次
4. `contextLoaded`：ApplicationContext创建并加载之后并在refresh之前调用
5. `finished`：run方法结束之前调用

该接口目前只有一个实现类：`EventPublishingRunListener`，在`spring.factories`中可以找到确实加载的是这个类，直接看`EventPublishingRunListener`：

```java
	private final SimpleApplicationEventMulticaster initialMulticaster;

	// 构造方法拿到了springApplication实例，启动参数
	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
      	// 事件的广播者
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
        // 向广播添加准备阶段实例化的监听器
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}

	@Override
	@SuppressWarnings("deprecation")
	public void starting() {
      	// 构造事件，使用事件广播者发送事件
		this.initialMulticaster
				.multicastEvent(new ApplicationStartedEvent(this.application, this.args));
	}
	
	// 其他接口方法分别构造不同事件，然后广播
```

整个事件监听者实现的类图：

![SpringBootSpringApplicationRunListener.png](https://ooo.0o0.ooo/2017/06/23/594cbdc2572ef.png)

从调用联系上，顺序如下：

![SpringBootSpringApplicationRunListener.jpg](https://ooo.0o0.ooo/2017/06/23/594cbe5e3a355.jpg)

这就把上文提到的监听器与Spring Application生命周期结合到了一起。有了`SpringApplicationRunListeners`就间接的掌管了所有的所有的`ApplicationListener`了，想让他们干什么只要调用对应`applicationListener`监听的时间就可以了。在启动逻辑中就分别在try-catch上面调用了`starting`，在try结尾处调用`finished`。

再回头看上文提到的监听器，才注意到监听器的实现除了继承`ApplicationListener`，还通过泛型监听指定的具体事件类型的。

例如上文提到默认的监听器中只有`LiquibaseServiceLocatorApplicationListener`响应`ApplicationStartingEvent` 事件：

```java
public class LiquibaseServiceLocatorApplicationListener
		implements ApplicationListener<ApplicationStartingEvent> {
	//...
	@Override
	public void onApplicationEvent(ApplicationStartingEvent event) {
		if (ClassUtils.isPresent("liquibase.servicelocator.ServiceLocator", null)) {
			new LiquibasePresent().replaceServiceLocator();
		}
	}
	//...
}
```

监听器的逻辑基本就是上面这些了。下文准备看看try里面容器启动的逻辑。
