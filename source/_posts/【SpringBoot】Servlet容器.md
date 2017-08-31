---
title: 【SpringBoot】Servlet容器
date: 2017-08-31 21:25:12
tags:
  - Java
  - Spring
categories: Spring
---

记得自己看 Spring Boot 源码的初衷是对部署时不需要额外的 Servlet 容器的好奇，好像看着看着关注到了其他细节。挖了几个坑就跳过了，今天把之前关于 Spring Boot 与 Servlet 容器的坑填一下。

先回（rang）顾（wo）一（xiang）下（xiang）之前与 Web 环境相关的内容：

- [SpringBoot包文件执行分析](http://zhangh.tk/2017/06/20/SpringBoot%E5%8C%85%E6%96%87%E4%BB%B6%E6%89%A7%E8%A1%8C%E5%88%86%E6%9E%90/)，这里说到开启了新线程调用应用入口，该线程设置了上下文类加载器 LaunchedURLClassLoader
- [【SpringBoot】SpringApplication实例创建](http://zhangh.tk/2017/07/05/%E3%80%90SpringBoot%E3%80%91SpringApplication%E5%AE%9E%E4%BE%8B%E5%88%9B%E5%BB%BA/)，这里说到根据 CLASSPTH 的包含的类推断当前是否是 web 环境
- [【SpringBoot】容器启动](http://zhangh.tk/2017/07/10/%E3%80%90SpringBoot%E3%80%91%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8/)，这里说到根据是否 web 环境创建做相关准备，创建处理 web 环境对应的 Spring容器
- [【Spring容器刷新】](http://zhangh.tk/2017/07/16/%E3%80%90Spring%E3%80%91%E5%AE%B9%E5%99%A8%E5%88%B7%E6%96%B0/)，这里说到 Spring 容器生命周期的相关方法

## Servlet 容器启动

[【SpringBoot】容器启动 ](http://zhangh.tk/2017/07/10/%E3%80%90SpringBoot%E3%80%91%E5%AE%B9%E5%99%A8%E5%90%AF%E5%8A%A8/)中提到的创建处理 web 环境对应的 Spring 容器实际创建的就是 AnnotationConfigEmbeddedWebApplicationContext，这个类是 ApplicationContext 的子类。其分别在 onRefresh 和 finishRefresh 方法中创建和启动 Servlet 容器：

```java
@Override
protected void onRefresh() {
   super.onRefresh();
   try {
      createEmbeddedServletContainer();
   }
   catch (Throwable ex) {
      throw new ApplicationContextException("Unable to start embedded container", ex);
   }
}

@Override
protected void finishRefresh() {
  super.finishRefresh();
  EmbeddedServletContainer localContainer = startEmbeddedServletContainer();
  if (localContainer != null) {
    publishEvent(
	new EmbeddedServletContainerInitializedEvent(this, localContainer));
  }
}
```

在 Spring 容器生命周期里与 Servlet 容器相关的逻辑封装在以上两个方法，分别用以创建和启动 Servlet 容器。

## Servlet 容器创建细节

上面说到 AnnotationConfigEmbeddedWebApplicationContext 中创建 Servlet 容器，具体的细节如下：

```java
// AnnotationConfigEmbeddedWebApplicationContext
private void createEmbeddedServletContainer() {
   EmbeddedServletContainer localContainer = this.embeddedServletContainer;
   ServletContext localServletContext = getServletContext();
   if (localContainer == null && localServletContext == null) {
      // 获取 Servlet 容器工厂
      EmbeddedServletContainerFactory containerFactory = getEmbeddedServletContainerFactory();
      // 获取 Servlet 容器初始化器
      // 使用 Servlet 容器初始化器创建 Servlet 容器
      this.embeddedServletContainer = 
        containerFactory.getEmbeddedServletContainer(getSelfInitializer());
   }
   else if (localServletContext != null) {
      try {
         getSelfInitializer().onStartup(localServletContext);
      }
      catch (ServletException ex) {
         throw new ApplicationContextException("Cannot initialize servlet context",
               ex);
      }
   }
   initPropertySources();
}
```

细节可以分为：

1. 获取 Servlet 容器工厂
2. 获取 Servlet 容器初始化器
3. 创建 Servlet 容器

### 获取 Servlet 容器工厂

```java
// AnnotationConfigEmbeddedWebApplicationContext

protected EmbeddedServletContainerFactory getEmbeddedServletContainerFactory() {
   // Use bean names so that we don't consider the hierarchy
   String[] beanNames = getBeanFactory()
         .getBeanNamesForType(EmbeddedServletContainerFactory.class);
   // 省略异常
   return getBeanFactory().getBean(beanNames[0],
         EmbeddedServletContainerFactory.class);
}
```

直接从 Spring 容器里获取 Servlet 容器工厂。一时之间卡住了不知道 EmbeddedServletContainerFactory 这个类时怎么注册到 Spring 容器中的。在网上查了一下发现是 META-INF/spring.factories 里指定了 EmbeddedServletContainerAutoConfiguration 这个配置类。而 EmbeddedServletContainerFactory 是 EmbeddedServletContainerAutoConfiguration 这个自动化配置类中被注册到 Spring 容器中的，关于配置类的注册我又要挖坑了。EmbeddedServletContainerAutoConfiguration：

```java
@AutoConfigureOrder(-2147483648)
@Configuration
@ConditionalOnWebApplication  // 在Web环境下起作用
@Import({EmbeddedServletContainerAutoConfiguration.BeanPostProcessorsRegistrar.class})
public class EmbeddedServletContainerAutoConfiguration {
    public EmbeddedServletContainerAutoConfiguration() {
    }
    
    // 在 import 中导入该类
    // 主要作用是注册 EmbeddedServletContainerCustomizerBeanPostProcessor,
    // ErrorPageRegistrarBeanPostProcessor
    public static class BeanPostProcessorsRegistrar 
      implements ImportBeanDefinitionRegistrar, BeanFactoryAware {
        // 省略
        public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                            BeanDefinitionRegistry registry) {
            if(this.beanFactory != null) {
                // 注册 EmbeddedServletContainerCustomizerBeanPostProcessor
                this.registerSyntheticBeanIfMissing(
                  registry, "embeddedServletContainerCustomizerBeanPostProcessor", 
                  EmbeddedServletContainerCustomizerBeanPostProcessor.class);
                // 注册 ErrorPageRegistrarBeanPostProcessor
                this.registerSyntheticBeanIfMissing(
                  registry, "errorPageRegistrarBeanPostProcessor", 
                  ErrorPageRegistrarBeanPostProcessor.class);
            }
        }

        private void registerSyntheticBeanIfMissing(BeanDefinitionRegistry registry, 
                                                    String name, Class<?> beanClass) {
            if(ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(beanClass, 
                                                                        true, false))) {
                RootBeanDefinition beanDefinition = new RootBeanDefinition(beanClass);
                beanDefinition.setSynthetic(true);
                registry.registerBeanDefinition(name, beanDefinition);
            }

        }
    }

    // 省略 Jetty，Undertow 配置

    @Configuration
    @ConditionalOnClass({Servlet.class, Tomcat.class})
    @ConditionalOnMissingBean(
        value = {EmbeddedServletContainerFactory.class},
        search = SearchStrategy.CURRENT
    )
    public static class EmbeddedTomcat {
        public EmbeddedTomcat() {
        }

        @Bean
        public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
            // 注册 EmbeddedServletContainerFactory 的实现类
            return new TomcatEmbeddedServletContainerFactory();
        }
    }
}
```

当满足特定条件时会注册具体的 EmbeddedServletContainerFactory 实现类，例如 TomcatEmbeddedServletContainerFactory。

这里还看到 注册了 EmbeddedServletContainerCustomizerBeanPostProcessor 和 ErrorPageRegistrarBeanPostProcessor 。简单看了一下 EmbeddedServletContainerCustomizerBeanPostProcessor，这是一个基本的 BeanPostProcessor，具体作用是对 EmbeddedServletContainerCustomizer 的实例进行定制，具体的实现包括：ErrorPageCustomizer，TomcatWebSocketContainerCustomizer，ServerProperties 等。

```java
// EmbeddedServletContainerCustomizerBeanPostProcessor 
public class EmbeddedServletContainerCustomizerBeanPostProcessor
      implements BeanPostProcessor, BeanFactoryAware {

   private ListableBeanFactory beanFactory;

   private List<EmbeddedServletContainerCustomizer> customizers;

   @Override
   public Object postProcessBeforeInitialization(Object bean, String beanName)
         throws BeansException {
      if (bean instanceof ConfigurableEmbeddedServletContainer) {
         // 处理ConfigurableEmbeddedServletContainer类型的bean
         postProcessBeforeInitialization((ConfigurableEmbeddedServletContainer) bean);
      }
      return bean;
   }

   private void postProcessBeforeInitialization(
         ConfigurableEmbeddedServletContainer bean) {
      // 对ConfigurableEmbeddedServletContainer类型的bean定制化处理
      for (EmbeddedServletContainerCustomizer customizer : getCustomizers()) {
         customizer.customize(bean);
      }
   }

   private Collection<EmbeddedServletContainerCustomizer> getCustomizers() {
      if (this.customizers == null) {
         // 从容器中找出所有EmbeddedServletContainerCustomizer类型的bean
         this.customizers = new ArrayList<EmbeddedServletContainerCustomizer>(
               this.beanFactory.getBeansOfType(EmbeddedServletContainerCustomizer.class,
                                               false, false).values());
         Collections.sort(this.customizers, AnnotationAwareOrderComparator.INSTANCE);
         this.customizers = Collections.unmodifiableList(this.customizers);
      }
      return this.customizers;
   }

}
```

### Servlet 容器初始化器

再回顾一下 Servlet 容器的创建核心逻辑：

```java
containerFactory.getEmbeddedServletContainer(getSelfInitializer());
```

先获取 Servlet 初始化器，然后创建 Servlet 容器

```java
private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
   return new ServletContextInitializer() {
      @Override
      public void onStartup(ServletContext servletContext) throws ServletException {
         selfInitialize(servletContext);
      }
   };
}

private void selfInitialize(ServletContext servletContext) throws ServletException {
   // Servlet 容器准备
   prepareEmbeddedWebApplicationContext(servletContext);
   // 初始 scopes
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   ExistingWebApplicationScopes existingScopes = new ExistingWebApplicationScopes(
         beanFactory);
   WebApplicationContextUtils.registerWebApplicationScopes(beanFactory,
         getServletContext());
   existingScopes.restore();
   // 注册 web 相关 bean
   WebApplicationContextUtils.registerEnvironmentBeans(beanFactory,
         getServletContext());
   // 初始化 servlet, filter, Listener 并注册
   for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
      beans.onStartup(servletContext);
   }
}

// 获取 ServletContext 初始化 bean
protected Collection<ServletContextInitializer> getServletContextInitializerBeans() {
	new ServletContextInitializerBeans(getBeanFactory());
}
```

```java
public ServletContextInitializerBeans(ListableBeanFactory beanFactory) {
   this.initializers = new LinkedMultiValueMap<Class<?>, ServletContextInitializer>();
   // 添加 ServletContextInitializer 类型的 bean
   addServletContextInitializerBeans(beanFactory);
   addAdaptableBeans(beanFactory);
   List<ServletContextInitializer> sortedInitializers = 
     new ArrayList<ServletContextInitializer>();
   for (Map.Entry<?, List<ServletContextInitializer>> entry : this.initializers.entrySet()) {
      AnnotationAwareOrderComparator.sort(entry.getValue());
      sortedInitializers.addAll(entry.getValue());
   }
   this.sortedList = Collections.unmodifiableList(sortedInitializers);
}

private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
   for (Entry<String, ServletContextInitializer> initializerBean : getOrderedBeansOfType(
         beanFactory, ServletContextInitializer.class)) {
      addServletContextInitializerBean(initializerBean.getKey(),
            initializerBean.getValue(), beanFactory);
   }
}

private void addServletContextInitializerBean(String beanName,
      ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
   if (initializer instanceof ServletRegistrationBean) {
      // servlet
      Servlet source = ((ServletRegistrationBean) initializer).getServlet();
      addServletContextInitializerBean(Servlet.class, beanName, initializer,
            beanFactory, source);
   }
   else if (initializer instanceof FilterRegistrationBean) {
      // filter
      Filter source = ((FilterRegistrationBean) initializer).getFilter();
      addServletContextInitializerBean(Filter.class, beanName, initializer,
            beanFactory, source);
   }
   else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
      // filter
      String source = ((DelegatingFilterProxyRegistrationBean) initializer)
            .getTargetBeanName();
      addServletContextInitializerBean(Filter.class, beanName, initializer,
            beanFactory, source);
   }
   else if (initializer instanceof ServletListenerRegistrationBean) {
      // listener
      EventListener source = ((ServletListenerRegistrationBean<?>) initializer)
            .getListener();
      addServletContextInitializerBean(EventListener.class, beanName, initializer,
            beanFactory, source);
   }
   else {
      addServletContextInitializerBean(ServletContextInitializer.class, beanName,
            initializer, beanFactory, initializer);
   }
}
```

### 创建 Servlet 容器

以 TomcatEmbeddedServletContainerFactory 的创建容器方法为例：

```java
@Override
public EmbeddedServletContainer getEmbeddedServletContainer(
      ServletContextInitializer... initializers) {
   Tomcat tomcat = new Tomcat();
   File baseDir = (this.baseDirectory != null ? this.baseDirectory
         : createTempDir("tomcat"));
   tomcat.setBaseDir(baseDir.getAbsolutePath());
   Connector connector = new Connector(this.protocol);
   tomcat.getService().addConnector(connector);
   customizeConnector(connector);
   tomcat.setConnector(connector);
   tomcat.getHost().setAutoDeploy(false);
   configureEngine(tomcat.getEngine());
   for (Connector additionalConnector : this.additionalTomcatConnectors) {
      tomcat.getService().addConnector(additionalConnector);
   }
   prepareContext(tomcat.getHost(), initializers);
   return getTomcatEmbeddedServletContainer(tomcat);
}
```

上面的代码完成了 Servlet 容器启动前所有的创建，配置动作。

## Servlet 容器启动

```java
private EmbeddedServletContainer startEmbeddedServletContainer() {
   EmbeddedServletContainer localContainer = this.embeddedServletContainer;
   if (localContainer != null) {
      localContainer.start();
   }
   return localContainer;
}
```

启动的逻辑并不复杂，直接调用 Servlet 容器的 start 方法。

## 总结

Spring Boot 使用的内嵌 Servlet 容器启动过程：

1. 从 spring.factories 中读取并注册了 EmbeddedServletContainerAutoConfiguration 类
2. EmbeddedServletContainerAutoConfiguration 注册了 Servlet 容器工厂类
3. 在 Spring 容器生命周期的 onRefresh 方法中开始创建 Servlet 容器
   1. 获取 Servlet 容器工厂
   2. 获取 Servlet 相关初始化 bean
   3. 配置并创建 Servlet 容器
4. 在 Spring 容器生命周期的 finishRefresh 方法中调用 Servlet 容器的 start 方法启动容器


