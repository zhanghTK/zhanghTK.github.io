---
title: 【Spring】容器刷新
date: 2017-07-16 00:20:31
tags:
  - Java
  - Spring
categories: Spring
---
前面文章在分析 Spring Boot 的启动过程中提到了 AbstractApplicationContext 类的 refresh 方法，这个方法是 Spring 容器的核心方法之一，该方法提供了基本的模板，完成容器的初始化，并提供了丰富的接口可以从纵向（容器启动过程）、横向（不同的容器子类型）、不同形式（声明式、编程式等）进行自定义扩展。

有关 ApplicationContext 和 BeanFactory 的内容可以参考之前关于 Spring 的文章：

- [Spring之BeanFactory](http://zhangh.tk/2017/03/13/Spring%E4%B9%8BBeanFactory/)
- [Spring之ApplicationContext](http://zhangh.tk/2017/03/19/Spring%E4%B9%8BApplicationContext/)

这里详细过一下 refresh 方法的过程，留个印象，方便以后回顾。

这篇文章就不记录关于 Spring Boot 在启动刷新阶段的特殊实现了，只关心 Spring 的通用实现模板

## 概览

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // 刷新前准备工作
      prepareRefresh();

      // 调用子类refreshBeanFactory()方法，获取bean factory
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      // 创建bean Factory的通用设置
      prepareBeanFactory(beanFactory);

      try {
         // 子类特殊的bean factory设置
         postProcessBeanFactory(beanFactory);

         // 实例化beanFactoryPostProcessor
         // 调用beanFactoryPostProcessor修改bean definition
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册 bean pst processors
         registerBeanPostProcessors(beanFactory);

         // 初始化信息源，和国际化相关
         initMessageSource();

         // 初始化容器事件传播器
         initApplicationEventMulticaster();

         // 调用子类特殊的刷新逻辑
         onRefresh();

         // 为事件传播器注册事件监听器
         registerListeners();

         // 实例化所有非懒加载单例
         finishBeanFactoryInitialization(beanFactory);

         // 初始化容器的生命周期事件处理器，并发布容器的生命周期事件
         finishRefresh();
      }

      catch (BeansException ex) {
         // ...
      }
      finally {
         // ...
      }
   }
}
```

## 刷新过程详述

### prepareRefresh

```java
protected void prepareRefresh() {
   // 设置Spring容器的启动时间，撤销关闭状态，开启活跃状态
   this.startupDate = System.currentTimeMillis();
   this.closed.set(false);
   this.active.set(true);

   if (logger.isInfoEnabled()) {
      logger.info("Refreshing " + this);
   }

   // 模板方法，设置属性源信息
   initPropertySources();

   // 验证环境信息里一些必须存在的属性
   getEnvironment().validateRequiredProperties();

   this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
}
```

### obtainFreshBeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 模板方法
   // 获取刷新Spring上下文的Bean工厂
   refreshBeanFactory();
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```

### prepareBeanFactory

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
   // 设置classloader
   beanFactory.setBeanClassLoader(getClassLoader());
   // 设置表达式解析器
   beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
   // 添加属性编辑注册器
   beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

   // 添加ApplicationContextAwareProcessor这个BeanPostProcessor用于回调
   // ApplicationContextAwareProcessor主要的作用是给实现XxxxAware的bean注入相关属性
   beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
   // 取消以下接口的自动注入
   beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
   beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
   beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
   beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
   beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

   // 设置特殊的类型对应的bean，修正依赖
   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
   beanFactory.registerResolvableDependency(ApplicationContext.class, this);

   // Register early post-processor for detecting inner beans as ApplicationListeners.
   beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

   // 如果自定义的Bean中没有一个名为”loadTimeWeaver”的Bena
   // 添加一个LoadTimeWeaverAwareProcessor
   if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      // Set a temporary ClassLoader for type matching.
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }

   // 如果自定义的Bean中没有名为”systemProperties”和”systemEnvironment”的Bean
   // 注册两个Bena，Key为”systemProperties”和”systemEnvironment”，Value为Map
   // 这两个bean包含系统配置和系统环绕信息
   if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
   }
   if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
      beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
   }
}
```

### postProcessBeanFactory

空实现，由具体子类实现。为子类特殊的 Application Context 实现指定特殊的 bean post 事件处理器

### invokeBeanFactoryPostProcessors

整个 Spring 流程中非常重要的方法，大量的用户自定义扩展都与这个方法有关。

第一次读的时候看着怪复杂的直接跳过去了，结果启动过程的好多问题还是没搞清楚。

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   // 实例化并调用所有注册的 beanFactoryPostProcessor
   PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

   // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
   // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
   if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
      beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
      beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
   }
}
```

复杂的东西不在上面，是 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors 方法的调用

说明这个方法之前有两个类需要特备说明，这个方法的操作都是围绕这两个类的：

- BeanFactoryPostProcessor

  在 bean 实例化阶段开始之前，对注册到容器的 BeanDefinition 保存的原始数据做出修改

- BeanDefinitionRegistryPostProcessor

  继承自上面的 BeanFactoryPostProcessor

  专门用来对注册的容器的 BeanFactoryPostProcessor 的BeanDefinition 保存的原始数据做出修改

有点拗口，不过意思类似指针与二级指针的关系。

整个 PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors 的实现太复杂了，但其实就一个意思：

1. 先用 BeanDefinitionRegistryPostProcessor 修改 BeanFactoryPostProcessor 的 BeanDefinition 
2. 然后用 BeanFactoryPostProcessor 修改普通的 BeanDefinition

具体在操作过程中 BeanDefinitionRegistryPostProcessor 和 BeanFactoryPostProcessor 执行的顺序有明确的限制，两者都是根据以下规则执行：

1. 首先执行实现 PriorityOrdered 接口的，调用顺序是按照接口方法 getOrder() 的实现
2. 其次执行实现 Ordered 接口的，调用顺序是按照接口方法 getOrder() 的实现
3. 最后执行既没有实现 PriorityOrdered  也没有实现 Ordered 的，调用顺序是 bean 的定义顺序

### registerBeanPostProcessors

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
   // 从Spring容器中找出的BeanPostProcessor接口的bean，并设置到BeanFactory的属性中
   PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

思路与 invokeBeanFactoryPostProcessors 方法类似。

但只是从容器中找出实现 BeanPostProcessor 的bean，排序后设置到BeanFactory的属性中，并不调用。

具体的排序过程：

1. 优先调用 PriorityOrdered 接口的子接口，调用顺序依照接口方法getOrder的返回值从小到大排序
2. 其次调用Ordered接口的子接口，调用顺序依照接口方法getOrder的返回值从小到大排序
3. 接着按照 BeanPostProcessor 实现类在配置文件中定义的顺序进行调用
4. 最后调用MergedBeanDefinitionPostProcessor接口的实现Bean，同样按照在配置文件中定义的顺序进行调用

### initMessageSource

```java
protected void initMessageSource() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
      // 如果自定义了名为”messageSource”的Bean
      // 直接实例化Bean，该Bean必须是MessageSource接口的实现Bean
      // 该Bean如果是HierarchicalMessageSource接口的实现类
      // 强转为HierarchicalMessageSource接口，并设置一下parentMessageSource
      this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
      // Make MessageSource aware of parent MessageSource.
      if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
         HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
         if (hms.getParentMessageSource() == null) {
            // Only set parent context as parent MessageSource if no parent MessageSource
            // registered already.
            hms.setParentMessageSource(getInternalParentMessageSource());
         }
      }
      if (logger.isDebugEnabled()) {
         logger.debug("Using MessageSource [" + this.messageSource + "]");
      }
   }
   else {
      // 如果没有自定义名为”messageSource”的Bean，那么会默认注册一个DelegatingMessageSource并加入
      // Use empty MessageSource to be able to accept getMessage calls.
      DelegatingMessageSource dms = new DelegatingMessageSource();
      dms.setParentMessageSource(getInternalParentMessageSource());
      this.messageSource = dms;
      beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
               "': using default [" + this.messageSource + "]");
      }
   }
}
```

### initApplicationEventMulticaster

```java
protected void initApplicationEventMulticaster() {
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
      // 如果自定义了名为”applicationEventMulticaster”的Bean
      // 实例化自定义的Bean，自定义的Bean必须是ApplicationEventMulticaster接口的实现类
      this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
      if (logger.isDebugEnabled()) {
         logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
      }
   }
   else {
      // 如果没有自定义名为”ApplicationEventMulticaster”的Bean
      // 注册一个类型为SimpleApplicationEventMulticaster的Bean
      this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
      beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
      if (logger.isDebugEnabled()) {
         logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
               APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
               "': using default [" + this.applicationEventMulticaster + "]");
      }
   }
}
```

### onRefresh

空的模板方法，根据不同的 ApplicationContext 子类实现对应的业务逻辑

### registerListeners

```java
protected void registerListeners() {
   // Register statically specified listeners first.
   for (ApplicationListener<?> listener : getApplicationListeners()) {
      getApplicationEventMulticaster().addApplicationListener(listener);
   }

   // Do not initialize FactoryBeans here: We need to leave all regular beans
   // uninitialized to let post-processors apply to them!
   String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
   for (String listenerBeanName : listenerBeanNames) {
      getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
   }

   // Publish early application events now that we finally have a multicaster...
   Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
   this.earlyApplicationEvents = null;
   if (earlyEventsToProcess != null) {
      for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
         getApplicationEventMulticaster().multicastEvent(earlyEvent);
      }
   }
}
```

### finishBeanFactoryInitialization

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
   //有容器转换服务bean时先实例化这种Bean
   if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
         beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
      beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
   }
   
   // 注册一个默认的嵌入式值解析器
   if (!beanFactory.hasEmbeddedValueResolver()) {
      beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
         @Override
         public String resolveStringValue(String strVal) {
            return getEnvironment().resolvePlaceholders(strVal);
         }
      });
   }

   // 实例化LoadTimeWeaverAware类型Bean
   String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
   for (String weaverAwareName : weaverAwareNames) {
      getBean(weaverAwareName);
   }

   // 停止使用临时类加载器
   beanFactory.setTempClassLoader(null);

   // 缓存容器中所有注册的BeanDefinition元数据，以防被修改
   beanFactory.freezeConfiguration();

   // 实例化剩余的所有非lazy-init singleton Bean
   beanFactory.preInstantiateSingletons();
}
```

### finishRefresh

```java
protected void finishRefresh() {
   // 初始化生命周期处理器（LifecycleProcessor）
   // 先找一下有没有自定义名为lifecycleProcessor的Bean，有的话就实例化出来
   // 该Bean必须是LifecycleProcessor的实现类
   // 没有则向Spring上下文中注册一个类型为DefaultLifecycleProcessor的LifecycleProcessor实现类
   initLifecycleProcessor();

   // 调用生命周期处理器的onRefresh方法
   getLifecycleProcessor().onRefresh();

   // 发布ContextRefreshedEvent事件告知对应的ApplicationListener进行响应的操作
   publishEvent(new ContextRefreshedEvent(this));

   // 如果设置了JMX相关的属性，则就调用该方法
   LiveBeansView.registerApplicationContext(this);
}
```
以上就是简单的记录了 Spring 启动过程中容器刷新的动作。

---

以上内容除了源码的阅读还参考了其他博客，学习到了很多细节，这里表示感谢了：

[SpringBoot源码分析之Spring容器的refresh过程](http://fangjian0423.github.io/2017/05/10/springboot-context-refresh/)

[Spring-- Ioc 容器Bean实例化的几种场景](http://blog.csdn.net/bubaxiu/article/details/41415685)

[《Spring技术内幕》学习笔记6——IoC容器的高级特性](http://blog.csdn.net/chjttony/article/details/6278627)

[Spring源码分析：非懒加载的单例Bean初始化前后的一些操作](http://www.importnew.com/24282.html)
