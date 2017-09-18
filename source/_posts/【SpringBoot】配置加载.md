---
title: 【SpringBoot】配置加载
date: 2017-09-18 22:14:15
tags:
  - Java
  - Spring
categories: Spring
---

前面的文章已经基本讲清楚从 Spring Boot 应用完整的启动过程了，不过中间过程缺少自动配置相关的功能实现说明。

我把功能配置分为两个部分：

- 自动化配置
- 条件注解

以上两个功能极大的简化了 Spring Boot 应用的配置

## 自动化配置

要弄清自动化配置的起源就得从入口类的 SpringBootApplication 注解说起：

```java
@EnableAutoConfiguration
public @interface SpringBootApplication {
  // 省略注解的字段，别名
}
```

SpringBootApplication 作为一个组合注解，配置的功能是其核心功能，这部分功能由 EnableAutoConfiguration 注解实现。 EnableAutoConfiguration  也是个符合注解：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({EnableAutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

EnableAutoConfiguration 的关键是导入了 EnableAutoConfigurationImportSelector ，这类继承自 AutoConfigurationImportSelector。在 Spring 容器刷新的过程中的 invokeBeanFactoryPostProcessors 方法会调用到 EnableAutoConfigurationImportSelector 的逻辑：

```java
// EnableAutoConfigurationImportSelector
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
   if (!isEnabled(annotationMetadata)) {
      return NO_IMPORTS;
   }
   try {
      // 获取注解属性
      AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
            .loadMetadata(this.beanClassLoader);
      AnnotationAttributes attributes = getAttributes(annotationMetadata);
      // 读取spring.factories属性文件中的数据
      List<String> configurations = getCandidateConfigurations(annotationMetadata,
            attributes);
      // 删除重复的配置类
      configurations = removeDuplicates(configurations);
      configurations = sort(configurations, autoConfigurationMetadata);
      // 找到@EnableAutoConfiguration注解中定义的需要被过滤的配置类
      Set<String> exclusions = getExclusions(annotationMetadata, attributes);
      checkExcludedClasses(configurations, exclusions);
      // 删除这些需要被过滤的配置类
      configurations.removeAll(exclusions);
      configurations = filter(configurations, autoConfigurationMetadata);
      fireAutoConfigurationImportListeners(configurations, exclusions);
      // 返回最终得到的自动化配置类
      return configurations.toArray(new String[configurations.size()]);
   }
   catch (IOException ex) {
      throw new IllegalStateException(ex);
   }
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
      AnnotationAttributes attributes) {
   // 调用SpringFactoriesLoader的loadFactoryNames静态方法
   // getSpringFactoriesLoaderFactoryClass方法返回的是EnableAutoConfiguration类对象
   List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
         getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
   Assert.notEmpty(configurations,
         "No auto configuration classes found in META-INF/spring.factories. If you "
               + "are using a custom packaging, make sure that file is correct.");
   return configurations;
}
```

EnableAutoConfigurationImportSelector 会使用 SpringFactoriesLoader 加载相应的配置

```java
// SpringFactoriesLoader 
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
   // 解析出properties文件中需要的key值
   String factoryClassName = factoryClass.getName();
   try {
      // 常量FACTORIES_RESOURCE_LOCATION的值为META-INF/spring.factories
      // 使用类加载器找META-INF/spring.factories资源
      Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
            ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
      List<String> result = new ArrayList<String>();
      while (urls.hasMoreElements()) {
         URL url = urls.nextElement();
         Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
         String factoryClassNames = properties.getProperty(factoryClassName);
         result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
      }
      return result;
   }
   catch (IOException ex) {
      throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
            "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
   }
}
```

## 条件注解

### 条件注解实现

以常见的 ConditionalOnClass 注解为例：

```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
   Class<?>[] value() default {};  // 要匹配的类
   String[] name() default {};  // 要匹配的类名
}
```

必要重要的是 Conditional 注解：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Conditional {
   Class<? extends Condition>[] value();
}
```

从命名可以看出 Conditional 注解的  value 属性是具体的 Condition 实现，Spring Boot 提供了通用的 Condition 抽象类： SpringBootCondition，其完成了具体匹配逻辑之外的其他模板：

```java
// SpringBootCondition

@Override
public final boolean matches(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   // 获得类名或方法名
   String classOrMethodName = getClassOrMethodName(metadata);
   try {
      // 抽象方法由子类实现匹配逻辑
      // 返回结构包装了匹配结果和log信息
      ConditionOutcome outcome = getMatchOutcome(context, metadata);
      // log
      logOutcome(classOrMethodName, outcome);
      // 报告一下
      recordEvaluation(context, classOrMethodName, outcome);
      // 返回匹配结果
      return outcome.isMatch();
   }
   catch // 省略异常处理
}
```

具体到 ConditionalOnClass 使用的是 OnClassCondition：

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   ClassLoader classLoader = context.getClassLoader();
   ConditionMessage matchMessage = ConditionMessage.empty();
   // 获得 ConditionalOnClass 注解的属性
   List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
   if (onClasses != null) {
      // 属性不为空
      // 获取具体缺失的类
      List<String> missing = getMatches(onClasses, MatchType.MISSING, classLoader);
      if (!missing.isEmpty()) {
         return ConditionOutcome
               .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                     .didNotFind("required class", "required classes")
                     .items(Style.QUOTE, missing));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
            .found("required class", "required classes").items(Style.QUOTE,
                  getMatches(onClasses, MatchType.PRESENT, classLoader));
   }
   // 获得 ConditionalOnMissingClass 注解的属性
   List<String> onMissingClasses = getCandidates(metadata,
         ConditionalOnMissingClass.class);
   if (onMissingClasses != null) {
      List<String> present = getMatches(onMissingClasses, MatchType.PRESENT,
            classLoader);
      if (!present.isEmpty()) {
         return ConditionOutcome.noMatch(
               ConditionMessage.forCondition(ConditionalOnMissingClass.class)
                     .found("unwanted class", "unwanted classes")
                     .items(Style.QUOTE, present));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
            .didNotFind("unwanted class", "unwanted classes").items(Style.QUOTE,
                  getMatches(onMissingClasses, MatchType.MISSING, classLoader));
   }
   // 返回全部匹配成功的ConditionalOutcome
   return ConditionOutcome.match(matchMessage);
}
```

### 条件注解激活

这里又要从 Spring 容器的 refresh 方法说起了，这个方法太重要了。其中的 invokeBeanFactoryPostProcessors 方法调用 BeanFactoryPostProcessor 和 BeanDefinitionRegistryPostProcessor 的处理逻辑，在 bean 实例化阶段开始之前，对注册到容器的 BeanDefinition 保存的原始数据做出修改。与条件注解激活有关的 posstProcessor 是 ConfigurationClassPostProcessor，其实现自 BeanFactoryPostProcessor 。

ConfigurationClassPostProcessor 会从 BeanFactory 中找出所有被 @Configuration 直接或间接注解的类（包括 @Component, @ComponentScan, @Import, @ImportResource 修饰的类）进行解析。对解析的结果使用 ConditionEvaluator 进行过滤判断。判断逻辑：

```java
public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
   // 
   if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
      return false;
   }

   if (phase == null) {
      if (metadata instanceof AnnotationMetadata &&
            ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
         return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
      }
      return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
   }

   List<Condition> conditions = new ArrayList<Condition>();
   for (String[] conditionClasses : getConditionClasses(metadata)) {
      for (String conditionClass : conditionClasses) {
         Condition condition = getCondition(conditionClass, this.context.getClassLoader());
         conditions.add(condition);
      }
   }

   AnnotationAwareOrderComparator.sort(conditions);

   for (Condition condition : conditions) {
      ConfigurationPhase requiredPhase = null;
      if (condition instanceof ConfigurationCondition) {
         requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
      }
      if (requiredPhase == null || requiredPhase == phase) {
         if (!condition.matches(this.context, metadata)) {
            return true;
         }
      }
   }

   return false;
}
```

其中 ConfigurationCondition 是 Condition 的子接口，内部主要定义了枚举类型表示条件注解生效的阶段：

```java
public interface ConfigurationCondition extends Condition {
   ConfigurationPhase getConfigurationPhase();
   enum ConfigurationPhase {
      PARSE_CONFIGURATION,
      REGISTER_BEAN
   }
}
```

完整的 ConfigurationClassPostProcessor 处理逻辑：

![configuration-annotation-process.png](https://i.loli.net/2017/08/29/59a4d381568f6.png)

---

参考：[SpringBoot源码分析之Spring容器的refresh过程](http://fangjian0423.github.io/2017/05/10/springboot-context-refresh/)
