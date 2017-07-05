---
title: 【SpringBoot】SpringApplication实例创建
date: 2017-07-05 21:22:54
tags:
  - Java
  - Spring
categories: Spring
---
书接上文，上回说到 Spring Boot 启动的入口是`JarLauncher`的`main`方法。其中的主要逻辑是在加载完各种资源后，开启一个新的线程调用程序的入口。整个应用的启动就此缓缓展开，本文只说明从`SpringApplication`的静态方法`run`调用后，生成`SpringApplication`实例的过程。而后的其他启动步骤不在本文记录。

上文说到`main`方法的调用：

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

静态的`run`方法是整个程序的入口，但最终还是实例化了`SpringApplication`对象：

```java
	private final Set<Object> sources = new LinkedHashSet<Object>();

	public static ConfigurableApplicationContext run(Object source, String... args) {
		return run(new Object[] { source }, args);
	}

	public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
		return new SpringApplication(sources).run(args);
	}

	public SpringApplication(Object... sources) {
		initialize(sources);
	}
```

今天的目标就是`initialize`方法的实现了：

```java
	private void initialize(Object[] sources) {
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
      	// 是否是web程序环境
		this.webEnvironment = deduceWebEnvironment();
        // 设置初始化器
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
        // 设置监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
      	// main方法所在类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

## 推断Web环境

先看看web环境的判断：

```java
	// org/springframework/boot/SpringApplication.java
	private static final String[] WEB_ENVIRONMENT_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
	
	private boolean deduceWebEnvironment() {
		for (String className : WEB_ENVIRONMENT_CLASSES) {
			if (!ClassUtils.isPresent(className, null)) {
				return false;
			}
		}
		return true;
	}
```

是通过判断给定的类加载器（null）是否可以加载给定的类（`WEB_ENVIRONMENT_CLASSES`）判断的。加载器给的是空，跟进去看到了为空时是有默认类加载器的：

```java
	// org/springframework/util/ClassUtils.java
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
		}
		if (cl == null) {
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				try {
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
				}
			}
		}
		return cl;
	}
```

## 查找 Main 方法

跳过其他方法先看`main`方法所在类的查找

```java
	private Class<?> deduceMainApplicationClass() {
		try {
			StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
			for (StackTraceElement stackTraceElement : stackTrace) {
				if ("main".equals(stackTraceElement.getMethodName())) {
					return Class.forName(stackTraceElement.getClassName());
				}
			}
		}
		catch (ClassNotFoundException ex) {
			// Swallow and continue
		}
		return null;
	}
```

这个`main`方法所在类的查找还是比较6的，万万没想到直接搞了个异常从堆栈里查找，也算是奇技淫巧吧。

## 初始化器，监听器的加载、实例化

下面重点看看`SpringApplication`实例化最重要的部分：初始化器和监听器的设置。

两者的核心逻辑都是一样，只是参数有所区别：

```java
	private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type) {
		return getSpringFactoriesInstances(type, new Class<?>[] {});
	}

	private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        // 使用上下文类加载器，加载指定配置文件中的配置
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
      	// 反射创建，没什么特别的
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```

```java
	// 加载的配置文件
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
	// 加载配置文件中key为factoryClass.getName()的项
	public static List<String> loadFactoryNames(Class<?> factoryClass, 
                                                ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		try {
			Enumeration<URL> urls = (classLoader != null ?
                                     classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                   ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			List<String> result = new ArrayList<String>();
			while (urls.hasMoreElements()) {
              	URL url = urls.nextElement();
				Properties properties = 
                  PropertiesLoaderUtils.loadProperties(new UrlResource(url));
				String factoryClassNames = 
                  properties.getProperty(factoryClassName);
              	result.addAll(Arrays.asList(
                  StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
			}
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
					"] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

所有的初始化器和监听器都是从 CLASSPATH 下的`META-INF/spring.factories`文件中获取的，具体配置：

```properties
# Application Context Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
org.springframework.boot.context.ContextIdApplicationContextInitializer,\
org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
org.springframework.boot.context.embedded.ServerPortInfoApplicationContextInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.ClearCachesApplicationListener,\
org.springframework.boot.builder.ParentContextCloserApplicationListener,\
org.springframework.boot.context.FileEncodingApplicationListener,\
org.springframework.boot.context.config.AnsiOutputApplicationListener,\
org.springframework.boot.context.config.ConfigFileApplicationListener,\
org.springframework.boot.context.config.DelegatingApplicationListener,\
org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener,\
org.springframework.boot.logging.ClasspathLoggingApplicationListener,\
org.springframework.boot.logging.LoggingApplicationListener
```

---

总结一下，SpringApplication的实例过程：

1. 判断是否是 web 环境
2. 加载初始化器和监听器并实例化
3. 查找 main 方法所在类

每个过程的特点：

1. 判断 web 环境是根据 Spring 的默认加载器是否能够加载给定类
2. 初始化器和监听器的加载都依据`META-INF/spring.factories`
3. main方法所在类的查找是抛出异常在堆栈中查找的
