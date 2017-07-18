---
title: 【SpringBoot】容器启动
date: 2017-07-10 20:24:11
tags:
  - Java
  - Spring
categories: Spring
---
Spring Boot 的启动前面说到了：

- 包文件启动：从`JarLauncher`的`main`方法启动，加载各种资源后，开启一个新的线程调用程序的`main`方法
- `SpringApplication`实例创建：判断是否是web环境，加载并实例化初始化器和监听器，查找`main`方法所在类
- 启动监听器：创建`SpringApplicationRunListeners`实例统一管理监听器。在启动过程中调用不同容器生命周期通知，创建不同的事件类型，由`ApplicationEventMulticaster`广播对应事件给`ApplicationListener`

前面做了各种准备工作就是为了最后的容器提供服务，本文以注释的形式记录一下 Spring Boot 容器的启动过程：

```java
	public ConfigurableApplicationContext run(String... args) {
		// ...
		try {
          	// 封装应用启动参数
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			// 准备环境
          	ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			Banner printedBanner = printBanner(environment);
          	// 创建容器
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
          	// 准备容器
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
          	// 刷新容器
			refreshContext(context);
          	// 运行runner
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
          	// ...
		}
	}
```

## 准备环境

```java
	private ConfigurableEnvironment prepareEnvironment(
			SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
      	// 创建环境
		ConfigurableEnvironment environment = getOrCreateEnvironment();
      	// 配置环境
		configureEnvironment(environment, applicationArguments.getSourceArgs());
      	// 监听器通知事件
		listeners.environmentPrepared(environment);
      	// 处理非web环境
		if (isWebEnvironment(environment) && !this.webEnvironment) {
			environment = convertToStandardEnvironment(environment);
		}
		return environment;
	}

	private ConfigurableEnvironment getOrCreateEnvironment() {
		if (this.environment != null) {
			return this.environment;
		}
		if (this.webEnvironment) {
			return new StandardServletEnvironment();
		}
		return new StandardEnvironment();
	}

	protected void configureEnvironment(ConfigurableEnvironment environment,
			String[] args) {
      	// 配置property
		configurePropertySources(environment, args);
        // 配置profiles
		configureProfiles(environment, args);
	}

	public static final String COMMAND_LINE_PROPERTY_SOURCE_NAME = "commandLineArgs";

	protected void configurePropertySources(ConfigurableEnvironment environment,
			String[] args) {
		MutablePropertySources sources = environment.getPropertySources();
      	// 判断defaultProperties是否为空
		if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
          	// MutablePropertySources内部维护一个list<PropertySource<?>>集合
          	// addLast先removeIfPresent()然后把defaultProperties加到最后
			sources.addLast(
					new MapPropertySource("defaultProperties", this.defaultProperties));
		}
      	// 判断addCommandLineProperties ==true 和 args的长度
		if (this.addCommandLineProperties && args.length > 0) {
          	// name=commandLineArgs
			String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
			if (sources.contains(name)) {
              	// 如果列表中包含，替换
				PropertySource<?> source = sources.get(name);
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(new SimpleCommandLinePropertySource(
						name + "-" + args.hashCode(), args));
				composite.addPropertySource(source);
				sources.replace(name, composite);
			}
			else {
              	// 如果列表中不包含
              	// 将配置信息添加在最前面，优先执行
				sources.addFirst(new SimpleCommandLinePropertySource(args));
			}
		}
	}

	protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {
		// spring.active.profile值
      	environment.getActiveProfiles();
		// But these ones should go first (last wins in a property key clash)
		Set<String> profiles = new LinkedHashSet<String>(this.additionalProfiles);
		profiles.addAll(Arrays.asList(environment.getActiveProfiles()));
      	// 将additionalProfiles和getActiveProfiles的值加到一起设置到环境中
		environment.setActiveProfiles(profiles.toArray(new String[profiles.size()]));
	}
```

注释比较详细的说明了代码的作用了：

1. 创建环境
2. 配置环境
3. 监听器通知环境准备完毕
4. 对异常环境处理

上面代码注释跳过了web环境相关的配置，猜测里面会有servlet容器相关配置，这里跳过另开一文记录。

## 创建容器

```java
public static final String DEFAULT_WEB_CONTEXT_CLASS = "org.springframework."
			+ "boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext";

public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
			+ "annotation.AnnotationConfigApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
   Class<?> contextClass = this.applicationContextClass;
   if (contextClass == null) {
      try {
         contextClass = Class.forName(this.webEnvironment
               ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
      }
      catch (ClassNotFoundException ex) {
         throw new IllegalStateException(
               "Unable create a default ApplicationContext, "
                     + "please specify an ApplicationContextClass",
               ex);
      }
   }
   return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```

这部分没什么可说的，就是根据环境通过反射直接创建对应的容器的实例。

## 准备容器

```java
private void prepareContext(ConfigurableApplicationContext context,
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments, Banner printedBanner) {
   context.setEnvironment(environment);
   postProcessApplicationContext(context);
   applyInitializers(context);
   listeners.contextPrepared(context);
   if (this.logStartupInfo) {
      logStartupInfo(context.getParent() == null);
      logStartupProfileInfo(context);
   }

   // Add boot specific singleton beans
   context.getBeanFactory().registerSingleton("springApplicationArguments",
         applicationArguments);
   if (printedBanner != null) {
      context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
   }

   // Load the sources
   Set<Object> sources = getSources();
   Assert.notEmpty(sources, "Sources must not be empty");
   load(context, sources.toArray(new Object[sources.size()]));
   listeners.contextLoaded(context);
}

protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
   //  beanNameGenerator不为空，注册
   if (this.beanNameGenerator != null) {
      context.getBeanFactory().registerSingleton(
            AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
            this.beanNameGenerator);
   }
   // resourceLoader不为空，注册
   if (this.resourceLoader != null) {
      if (context instanceof GenericApplicationContext) {
         ((GenericApplicationContext) context)
               .setResourceLoader(this.resourceLoader);
      }
      if (context instanceof DefaultResourceLoader) {
         ((DefaultResourceLoader) context)
               .setClassLoader(this.resourceLoader.getClassLoader());
      }
   }
}

protected void applyInitializers(ConfigurableApplicationContext context) {
   // 容器使用初始化器
   for (ApplicationContextInitializer initializer : getInitializers()) {
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
      initializer.initialize(context);
   }
}

// 加载各种bean到容器中
protected void load(ApplicationContext context, Object[] sources) {
   if (logger.isDebugEnabled()) {
      logger.debug(
            "Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
   }
   BeanDefinitionLoader loader = createBeanDefinitionLoader(
         getBeanDefinitionRegistry(context), sources);
   if (this.beanNameGenerator != null) {
      loader.setBeanNameGenerator(this.beanNameGenerator);
   }
   if (this.resourceLoader != null) {
      loader.setResourceLoader(this.resourceLoader);
   }
   if (this.environment != null) {
      loader.setEnvironment(this.environment);
   }
   loader.load();
}
```

容器的准备阶段的操作：

1. 为容器设置环境信息
2. 容器的预设置
3. 生效初始化器
4. 监听器通知容器准备完毕
5. 加载各种bean到容器中
6. 监听器通知容器加载完毕

## 刷新容器

```java
private void refreshContext(ConfigurableApplicationContext context) {
   refresh(context);
   if (this.registerShutdownHook) {
      try {
         context.registerShutdownHook();
      }
      catch (AccessControlException ex) {
         // Not allowed in some environments.
      }
   }
}

protected void refresh(ApplicationContext applicationContext) {
   Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
   ((AbstractApplicationContext) applicationContext).refresh();
}
```

最后这个`((AbstractApplicationContext) applicationContext).refresh();`就厉害了，终于把 Spring 容器整合到一起了，这里不详细分析，说起来话有点多，还是再开一篇文章介绍。

## Runner

```java
protected void afterRefresh(ConfigurableApplicationContext context,
      ApplicationArguments args) {
   callRunners(context, args);
}

private void callRunners(ApplicationContext context, ApplicationArguments args) {
   List<Object> runners = new ArrayList<Object>();
   // 找出Spring容器中ApplicationRunner接口的实现类
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
   // 找出Spring容器中CommandLineRunner接口的实现类
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
   AnnotationAwareOrderComparator.sort(runners);
   for (Object runner : new LinkedHashSet<Object>(runners)) {
      if (runner instanceof ApplicationRunner) {
         callRunner((ApplicationRunner) runner, args);
      }
      if (runner instanceof CommandLineRunner) {
         callRunner((CommandLineRunner) runner, args);
      }
   }
}

private void callRunner(ApplicationRunner runner, ApplicationArguments args) {
   try {
      (runner).run(args);
   }
   catch (Exception ex) {
      throw new IllegalStateException("Failed to execute ApplicationRunner", ex);
   }
}

private void callRunner(CommandLineRunner runner, ApplicationArguments args) {
   try {
      (runner).run(args.getSourceArgs());
   }
   catch (Exception ex) {
      throw new IllegalStateException("Failed to execute CommandLineRunner", ex);
   }
}
```

刷新完容器后，Spring 容器的启动就全部完成，最后这个`afterRefresh`方法是调用程序中的`ApplicationRunner`和`CommandLineRunner`，没有什么特别的。

---

总结一下，整个容器的启动步骤：

1. 准备环境
   1. 创建环境
   2. 配置环境
   3. 对异常环境处理
2. 创建容器
   1. 根据环境通过反射直接创建对应的容器的实例
3. 刷新容器
   1. `AbstractApplicationContext.refresh()`
4. 运行runner
   1. 调用程序中的`ApplicationRunner`和`CommandLineRunner`

监听器穿插在以上步骤中，根据容器的启动、运行状态广播对应的事件。

上述步骤当中忽略了两个关键的具体实现：

1. 容器的创建，根据环境创建容器：

   web环境：`AnnotationConfigEmbeddedWebApplicationContext`

   非web环境：``AnnotationConfigApplicationContext`

2. 容器的创建完成后的刷新

这两部分内容较多，挖坑以后填写。
