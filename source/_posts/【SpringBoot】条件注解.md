---
title: 【SpringBoot】条件注解
date: 2017-12-08 20:45:21
tags:
  - Java
  - Spring
categories: Spring
---
## 条件注解

### ConditionOnClass

以 ConditionOnClass 为例分析：

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {

   // The classes that must be present.
   Class<?>[] value() default {};

   // The classes names that must be present.
   String[] name() default {};

}
```

ConditionOnClass 注解使用 Conditional 注解修饰：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
  
   // All Conditions that must match in order for the component to be registered.
   Class<? extends Condition>[] value();
}
```

Conditional 定义了一个条件数组 Condition[]，条件的定义：

```java
public interface Condition {
   boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

看一下上面 Conditional 注解使用的 OnClassCondition。

首先其继承于 SpringBootCondition，SpringBoot 中条件注解都继承这个抽象类：

```java
public abstract class SpringBootCondition implements Condition {

   @Override
   public final boolean matches(ConditionContext context,
         AnnotatedTypeMetadata metadata) {
      // 得到类名或者方法名(条件注解可以作用的类或者方法上)
      String classOrMethodName = getClassOrMethodName(metadata);
      try {
         // 抽象方法，具体子类实现
         // ConditionOutcome记录了匹配结果boolean和log信息
         ConditionOutcome outcome = getMatchOutcome(context, metadata);
         // log记录一下匹配信息
         logOutcome(classOrMethodName, outcome);
         // 报告记录一下匹配信息
         recordEvaluation(context, classOrMethodName, outcome);
         return outcome.isMatch();
      }
      catch (...) {
        // ...
      }
   }

   // 省略其他
}
```

OnClassCondition 实现：

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   ClassLoader classLoader = context.getClassLoader();
   ConditionMessage matchMessage = ConditionMessage.empty();
   // 得到 @ConditionalOnClass 注解的类
   List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
   if (onClasses != null) {
      // 如果类存在
      // 得到在类加载器中不存在的类
      List<String> missing = getMatches(onClasses, MatchType.MISSING, classLoader);
      if (!missing.isEmpty()) {
         // 如果存在类加载器中不存在对应的类，返回一个匹配失败的 ConditionalOutcome
         return ConditionOutcome
               .noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                     .didNotFind("required class", "required classes")
                     .items(Style.QUOTE, missing));
      }
      // 如果类加载器中有所有的对应类，匹配信息进行记录
      matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
            .found("required class", "required classes").items(Style.QUOTE,
                  getMatches(onClasses, MatchType.PRESENT, classLoader));
   }
   // 对@ConditionalOnMissingClass注解做相同的逻辑处理
   // @ConditionalOnClass和@ConditionalOnMissingClass可以一起使用
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
   return ConditionOutcome.match(matchMessage);
}
```

### ConditionalOnBean

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
   // 匹配的bean类型
   Class<?>[] value() default {};

   // 匹配的bean类型的类名
   String[] type() default {};

   // 匹配的bean注解
   Class<? extends Annotation>[] annotation() default {};

   // 匹配的bean的名字
   String[] name() default {};

   // 搜索策略
   // CURRENT(只在当前容器中找)
   // PARENTS(只在所有的父容器中找；但是不包括当前容器)
   // ALL(CURRENT和PARENTS的组合)
   SearchStrategy search() default SearchStrategy.ALL;
}
```

OnBeanCondition 实现：

```java
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context,
      AnnotatedTypeMetadata metadata) {
   ConditionMessage matchMessage = ConditionMessage.empty();
   if (metadata.isAnnotated(ConditionalOnBean.class.getName())) {
      // 针对 @ConditionalOnBean 注解
      // 构造一个BeanSearchSpec，会从@ConditionalOnBean注解中获取属性，然后设置到BeanSearchSpec中
      BeanSearchSpec spec = new BeanSearchSpec(context, metadata,
            ConditionalOnBean.class);
      // 从BeanFactory中根据策略找出所有匹配的bean
      List<String> matching = getMatchingBeans(context, spec);
      if (matching.isEmpty()) {
         // 如果没有匹配的bean，返回一个没有匹配成功的ConditionalOutcome
         return ConditionOutcome.noMatch(
               ConditionMessage.forCondition(ConditionalOnBean.class, spec)
                     .didNotFind("any beans").atAll());
      }
      // 如果找到匹配的bean，匹配信息进行记录
      matchMessage = matchMessage.andCondition(ConditionalOnBean.class, spec)
            .found("bean", "beans").items(Style.QUOTE, matching);
   }
   if (metadata.isAnnotated(ConditionalOnSingleCandidate.class.getName())) {
      // 相同的逻辑，针对@ConditionalOnSingleCandidate注解
      BeanSearchSpec spec = new SingleCandidateBeanSearchSpec(context, metadata,
            ConditionalOnSingleCandidate.class);
      List<String> matching = getMatchingBeans(context, spec);
      if (matching.isEmpty()) {
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnSingleCandidate.class, spec)
               .didNotFind("any beans").atAll());
      }
      else if (!hasSingleAutowireCandidate(context.getBeanFactory(), matching,
            spec.getStrategy() == SearchStrategy.ALL)) {
         // 多了一层判断，判断是否只有一个bean
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnSingleCandidate.class, spec)
               .didNotFind("a primary bean from beans")
               .items(Style.QUOTE, matching));
      }
      matchMessage = matchMessage
            .andCondition(ConditionalOnSingleCandidate.class, spec)
            .found("a primary bean from beans").items(Style.QUOTE, matching);
   }
   if (metadata.isAnnotated(ConditionalOnMissingBean.class.getName())) {
      // 相同的逻辑，针对 @ConditionalOnMissingBean 注解
      BeanSearchSpec spec = new BeanSearchSpec(context, metadata,
            ConditionalOnMissingBean.class);
      List<String> matching = getMatchingBeans(context, spec);
      if (!matching.isEmpty()) {
         return ConditionOutcome.noMatch(ConditionMessage
               .forCondition(ConditionalOnMissingBean.class, spec)
               .found("bean", "beans").items(Style.QUOTE, matching));
      }
      matchMessage = matchMessage.andCondition(ConditionalOnMissingBean.class, spec)
            .didNotFind("any beans").atAll();
   }
   return ConditionOutcome.match(matchMessage);
}
```

## 条件注解激活

[【Spring】容器刷新 ](http://zhangh.tk/2017/07/16/%E3%80%90Spring%E3%80%91%E5%AE%B9%E5%99%A8%E5%88%B7%E6%96%B0/) 中 invokeBeanFactoryPostProcessors 方法会处理 BeanDefinitionRegistryPostProcessor 和 BeanDefinitionRegistryPostProcessor 的实现。其中 ConfigurationClassPostProcessor 是最低优先级执行的 BeanDefinitionRegistryPostProcessor。

ConfigurationClassPostProcessor 会使用 ConfigurationClassParser 的 ConditionEvaluator 解析所有 @Configuration 注解的bean。

ConfigurationClassParser 使用 ConditionEvaluator 解析：

```java
if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(),
                                       ConfigurationPhase.PARSE_CONFIGURATION)) {
  return;
}
```

ConditionEvaluator的 shouldSkip 方法：

```java
public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationPhase phase) {
   // 如果这个类没有被@Conditional注解所修饰，不会skip
   if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
      return false;
   }
   // 如果参数中沒有设置条件注解的生效阶段
   if (phase == null) {
      // 是配置类的话直接使用PARSE_CONFIGURATION阶段
      if (metadata instanceof AnnotationMetadata &&
            ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
         return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
      }
      // 否则使用REGISTER_BEAN阶段
      return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
   }
   // 获取配置类的条件注解得到条件数据，并添加到集合中
   List<Condition> conditions = new ArrayList<Condition>();
   for (String[] conditionClasses : getConditionClasses(metadata)) {
      for (String conditionClass : conditionClasses) {
         Condition condition = getCondition(conditionClass, this.context.getClassLoader());
         conditions.add(condition);
      }
   }
   // 对条件集合做个排序
   AnnotationAwareOrderComparator.sort(conditions);
   // 遍历条件集合
   for (Condition condition : conditions) {
      ConfigurationPhase requiredPhase = null;
      if (condition instanceof ConfigurationCondition) {
         requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
      }
      // 没有这个解析类不需要阶段的判断或者解析类和参数中的阶段一致才会继续进行
      if (requiredPhase == null || requiredPhase == phase) {
         // 不满足条件返回 true
         if (!condition.matches(this.context, metadata)) {
            return true;
         }
      }
   }
   return false;
}
```

条件注入激活还是在容器 refresh 过程中，更具体是在 invokeBeanFactoryPostProcessors 方法中解析配置类时，会跳过不满足条件的类。

---

注解条件使用的 demo：[SpringBoot源码分析之条件注解的底层实现](http://fangjian0423.github.io/2017/05/16/springboot-condition-annotation/)


