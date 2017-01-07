---
title: 【翻译】SpringData官方文档第四章
date: 2016-11-27
tags:
  - 翻译
categories: 翻译
---
说明：本翻译[4.6](http://ifeve.com/repositories-custom-implementations/)和[4.7](http://ifeve.com/spring-data-4-7/)段发布在[并发编程网](http://ifeve.com/),其他段落为了熟悉上下文而翻译，没有精校。首次参与翻译任务，翻译的不好请指正。

---

# 第四章 使用Spring Data Repositories

Spring Data repository abstraction目的是为了显著的简化必要的样板式代码来为多种持久化数据存储实现数据层。

> 重要
>
> 本章解释Spring Data repositories核心的概念和接口。Spring Data repository documentation 与 你的模块。本章这些信息是从Spring Data Commons模块获取的。它使用了JPA模块的配置和代码示范,命名引用覆盖XML配置，它支持所有的Spring Data模块支持的repository API，Repository查询关键字覆盖被repository 接口一般的关键字查询方法。你的模块特性的细节信息，查询文档对应模块。
>

## 4.1 核心概念

Spring Data repository抽象接口的核心是`Repository`（可能没有那么惊喜）。它需要管理实体类以及实体类的id作为参数。此接口主要用作是获取要使用的类型接口并帮助扩展这个接口。

`CrudRepository`为被管理的实体类提供复杂CRUD功能。

## 4.2 查询方法

标准的CRUD功能repositories通常有查询底层数据库。

在Spring Data中，分词声明这些查询变成了四个步骤的过程：

1. 声明一个接口继承`Repository`或者它的一个子类，并且指定要被处理的实体类和Id类型。

   ```java
   interface PersonRepository extends Repository<Person, Long> { … }
   ```


1. 在接口中声明查询方法。

   ```java
   interface PersonRepository extends Repository<Person, Long> {
     List<Person> findByLastname(String lastname);
   }
   ```


1. 创建Spring生成代理实现上面接口，通过JavaConfig：

   ```java
   import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
   @EnableJpaRepositories
   class Config {}
   ```

   或者通过XML配置：

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xmlns:jpa="http://www.springframework.org/schema/data/jpa"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd
   http://www.springframework.org/schema/data/jpa
   http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">
   <jpa:repositories base-package="com.acme.repositories"/>
   </beans>
   ```

   这个实例中使用了JPA命名空间。

   如果你为其他存储使用repository接口，你需要修改来适应你的存储模块命名空间的声明，大概就是替换jpa为期望的，例如mongodb。

   当然，注意JavaConfig并没有明确配置一个包默认使用注解的包。

   为了定制包被扫描可以使用数据存储注解`@Enable`的一个属性`basePackage`。


1. 获得repository实例注入并使用。

   ```java
   public class SomeClient {
     @Autowired
     private PersonRepository repository;
     public void doSomething() {
       List<Person> persons = repository.findByLastname("Matthews");
     }
   }
   ```


以下部分详细说明每个步骤。

## 4.3 定义repository接口

第一步定义一个实体类依赖repository接口。

这个接口必须继承`Repository`接口并且指定实体类型和Id类型。

如果你希望实体类型拥有CRUD方法，将`Repository`接口替换成继承`CrudRepository`。

### 4.3.1 小巧repository定义 

一般情况下，你的repository接口应该继承`Repository`，`CrudRepository`或者`PagingAndSortingRepository`

除此之外,如果你不想继承Spring Data接口,你也可以使用`@Repository`注解定义你的接口

继承`CrudRepository`提供了一系列完整的方法来操纵你的实体.

如果你希望选择方法,简单的从`CurdRepository`复制你希望获得的方法到你的`Repository`

> 注意
>
> 注意这允许你定义你自己的抽闲建立在Spring Data Repositories功能.

例5.有选择的展现`CRUD`方法

```java
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends Repository<T, ID> {
  T findOne(ID id);
  T save(T entity);
}

interface UserRepository extends MyBaseRepository<User, Long> {
  User findByEmailAddress(EmailAddress emailAddress);
}
```

这是第一步你定义一个通用的基本接口,接口供你所有的实体repositories使用并提供`findOne()`和`save()`方法

这些方法会被转发到你选择的Spring Data提供的基本repository实现,例如JPA`SimpleJpaRepository`,因为他们匹配`CrudRepository`的方法签名.

因此`UserPepository`现在可以保存用户,查找唯一用户根据id,并且触发一个查询去查找`Users`根据他们的邮箱地址

> 注意
>
> 注意,中间的repository接口使用了注解`NoRepositoryBean`.
>
> 对所有Spring Data在运行时不需要生成实例的repository接口,确保你为他们添加了注解.

### 4.3.2 在多个Spring Data模块使用Repositories

在你的应用中使用唯一的Spring Data模块,所有repository接口定义范围限制在Spring Data模块.

有时候应用需要使用不止一个Spring Data模块.

这种情况下,需要repository定义在持久化技术之间有所区别.

在class path发现多个repository工厂时,Spring Data严格检测repository配置模块.

在repository或者实体类严格配置需要得细节以决定Spring Data模块绑定一个repository的定义:

1. 如果repository定义继承模块特殊的repository,那么对Spring Data模块这是一个有效的备选.


1. 如果实体类被模块特有的注解类型注解,那么对Spring Data模块这是一个有效的备选.

   Spring Data模块接收第三方注解(例如 JPA的`@Entity`)或者提供自己的注解例如`@Document`为Spring Data MongoDB/Spring Data Elasticsearch.


例6.使用模块特有的接口定义Repository

```java
interface MyRepository extends JpaRepository<User, Long> { }
@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends JpaRepository<T,ID{
  …
}

interface UserRepository extends MyBaseRepository<User, Long> {
  …
}
```

`MyRepository`和`UserRepository`继承`JpaRepository`在他们类型层级.他们有效的表示了Spring Data JPA模块.

例7.使用通用的接口定义`Repository`.

```java
interface AmbiguousRepository extends Repository<User, Long> {
  …
}

@NoRepositoryBean
interface MyBaseRepository<T, ID extends Serializable> extends
  CrudRepository<T,ID> {
  …
}

interface AmbiguousUserRepository extends MyBaseRepository<User, Long> {
  …
}
```

`AmbiguousRepository`和`AmbiguousUserRepository`在它们继承体系里只继承了`Repository`和`CrudRepository`.使用唯一的Spring Data模块是这是完成正确的,多个模块不能识别`Repository`到底绑定哪个Spring Data.

例8.使用注解配合实体类定义`Repositor`

```java
interface PersonRepository extends Repository<Person, Long> {
  …
}

@Entity
public class Person {
  …
}

interface UserRepository extends Repository<User, Long> {
  …
}

@Document
public class User {
  …
}
```

`PersonRepository`引用使用了JPA的注解`@Entity`进行注解的`Person`,所以这个repository明确的属于Spring Data JPA.`UserRepository`使用了Spring Data MongoDB的注解`@Document`进行注解.

例9.使用混合注解配合实体类定义`Repositor`

```java
interface JpaPersonRepository extends Repository<Person, Long> {
  …
}

interface MongoDBPersonRepository extends Repository<Person, Long> {
  …
}

@Entity
@Document
public class Person {
  …
}
```

这个示例展示实体类同时使用JPA和Spring Data MongoDB注解.

这里定义了两个repository,`JpaPersonRepository`和`MongoDBPersonRepository`.

一个被JPA使用,另一个被MongoDB使用.

Spring Data不再能告诉repository区分,这将导致不清晰的行为

"使用模块特有的接口定义Repository"和"使用注解配合实体类定义`Repositor`"都使用了严格的repository配置为一个特定的Spring Data模块识别可选的repository


使用多种持久化技术特定的注解在同一个实体类上可能在锅中持久化技术上重用了实体类,但是这样做Spring Data将不能确定为repository绑定唯一的模块

最后一种方式区分repository是划分repository的基本包.


基本包定义起始点是扫描repository接口定义,这意味着在合适的包定义repository.

默认的,注解配置使用类配置的包

基于XML基本配置在这里.

例10.注解配置基本包

```java
@EnableJpaRepositories(basePackages = "com.acme.repositories.jpa")
@EnableMongoRepositories(basePackages = "com.acme.repositories.mongo")
interface Configuration { }
```



## 4.4 Defining query methods

## 4.4 定义查询方法

repository代理有两种方式获得通过方法名获得指定查询.可以通过方法名直接获得查询,或者手动定义查询.有可用的选项定义在实际的存储.这有一些策略决定实际查询如何被创建.让我们一些看看可用的选项.

### 4.4.1 Query lookup strategies

下面这些repository基本组件的决定查询可用的策略.你可以配置策略,在XML配置中通过命名空间`query-look-strategy`属性或者在Java配置中通过启用${store} Repository的属性注解`queryLookupStrategy`.一些策略可能不能支持特定的数据库.

- CREATE 试图通过方法名构建一个指定查询.一般处理是从方法名移除一系列给定的前缀并解析方法其他部分.更多查询构建信息阅读Query creation.
- USE_DECLARED_QUERY试图查找一个准确的查询,并在找不到时抛出一个异常.查询可以被通过注解定义或者其它方式定义.查询特殊存储的文档了解存储的可用选择.如果repository基本组件在启动时不能为方法找到一个准确的查询,将会失败
- CREATE_IF_NOT_FOUND(默认)联合了CREATE和USE_DECLARED_QUERY.首先查找一个准确查询,如果没有找到,创建一个定制方法基于名字的查询.这是默认的查询策略,因此如果你没有做任何明确配置.它允许根据方法名快速查询定义,而且定制查询根据需要引入声明的查询.


### 4.4.2 查询创建

建成Spring Data repository基本组件的查询构建机制有用的构建了repository所有实体类的约束查询.

这个机制分隔方法前缀`find...By`,`read...By`,`query...By`,`count...By`以及`get...By`,并解析其他部分.这个引入条款可以表达包含的特性例如`Distinct`,来设置明确的标志在要生成的查询上.然而,第一个by扮演了分解符来指明真实条件的起始.分词在一个非常基本的水平,你可以在实体属性上定义条件并且连接使用and或or连接他们.

例11.来自方法名的查询创建

```java
public interface PersonRepository extends Repository<User, Long> {
  List<Person> findByEmailAddressAndLastname(EmailAddress emailAddress, String
                                             lastname);
  // Enables the distinct flag for the query
  List<Person> findDistinctPeopleByLastnameOrFirstname(String lastname, String
                                                       firstname);
  List<Person> findPeopleDistinctByLastnameOrFirstname(String lastname, String
                                                       firstname);

  // Enabling ignoring case for an individual property
  List<Person> findByLastnameIgnoreCase(String lastname);
  // Enabling ignoring case for all suitable properties
  List<Person> findByLastnameAndFirstnameAllIgnoreCase(String lastname, String
                                                       firstname);
  // Enabling static ORDER BY for a query
  List<Person> findByLastnameOrderByFirstnameAsc(String lastname);
 
  List<Person> findByLastnameOrderByFirstnameDesc(String lastname);
}
```

解析方法的实际结果取决于你创建查询的持久化存储.然而,这有些问题需要注意:

- 表达式通常串行的连接属性的遍历与操作符.你可以使用and和or连接属性表达式.你也可以给属性表达式使用操作符,例如between,lessThan,granterThan,like.你可以使用的表达式操作符有between,LessThan,GreaterThan,Loke.被支持的操作符根据多样的数据库决定,因此查询你引用文档的恰当部分.
- 方法解析支持为单独属性设置一个IgnoreCase标志(例如,findByLastnameIgnoreCase(...))或者为所有属性都支持忽略情况(通常是String情形,例如,findByLastnameAndFirstnameAllIgnoreCase(…)).忽略情况是否被支持由多样的数据库决定，所以具体存储查询方法在引用文档查询相关章节.
- 你可以为查询方法引用的属性提供静态排序连接OrderBy子句并且提供排序方向(Asc或Desc).为创建一个查询方法支持动态排序,查看特殊参数处理.



### 4.4.3 属性表达式

属性表达式可以只为管理的实体类的直接属性使用，就像前面所展示的那样。查询常见时你已经确认解析的字段是管理的实体类的一个字段.然而,你也可以通过最近的字段定义一个约束.假设`Person`的`Address`有一个`ZipCode`字段.这种情况一个方法如果这样命名:

```java
List<Person> findByAddressZipCode(ZipCode zipCode);
```

创建属性遍历`x.address.zipCode`.决策算法从把全部(`AddressZipCode`)作为属性开始并且检查实体类依此命名的属性(小写开头).如果算法成功了,就是用这个属性.否则,算法将源码部分驼峰式大小写从右侧头部到尾巴分隔,并试图找到相应的属性,在我们的例子中,`AddressZip`和`Code`.如果从头部找到一个属性,算法将在这里生成树处理后面的部分,使用描述的方式分隔.如果第一个分隔没有匹配到,算法移动分割点到左侧继续(`Address`,`ZipCode`).

这在大多数情况都可以使用,但也可能选择错误的属性.假设`Person`类也有一个`addressZip`属性.算法将匹配第一个匹配的,本质上选择了错误属性,最终失败(伴随的`addressZip`类型没有属性`code`).

没了解决这种起义,你可以在方法名称内使用`_`手动的定义遍历点.所以我们的方法名称最终像这样:

```java
List<Person> findByAddress_ZipCode(ZipCode zipCode);
```

As we treat underscore as a reserved character we strongly advise to follow standard Java naming
conventions (i.e. not using underscores in property names but camel case instead) 

我们对待下划线当做一个保留关键字,我们强力建议遵循标准Java命名规范(例如,使用驼峰命名而不是下划线命名属性名称)

### 4.4.4 特殊参数处理

处理你查询中的参数,你可以定义简单的方法参数像上面的示例中.处理之外基本组件可以识别出某些特殊类型例如`Pageable`和`Sort`用来动态的编码和排序你的查询.

例12.在查询方法使用`Pageable`,`Slice`和`Sort`

```java
Page<User> findByLastname(String lastname, Pageable pageable);
Slice<User> findByLastname(String lastname, Pageable pageable);
List<User> findByLastname(String lastname, Sort sort);
List<User> findByLastname(String lastname, Pageable pageable);
```

第一个方法允许你在通过一个`org.springframework.data.domain.Pageable`实例在查询方法中动态添加分页信息在你的静态定义查询中.一个`Page`清楚数据的全部数量和页面总数.它是通过触发计数的基础设施查询计算总数。这个代价可能是昂贵的具体取决于所使用的存储,可以用`Slice`代替.一个`Slict`只知道下一个`Slice`可到到哪里,当运行在一个大的结果集上时这可能已经足够了.排序选项也通过`Pageable`实例处理.如果你只需要排序,简单起见添加`org.springframework.data.domain.Sort`参数在你的方法.如你所见,简单返回一个列表.在这种情况下,额外元数据构建实际的`Page`实例将不会被创建(反过来这意味着,没有发出额外的统计查询所必须的)简单约束只在给定的范围内查询.

> 注意
>
> 为了尽早得到你查询了多少页,你必须出发一个格外统计查询.默认的,这个查询你实际触发的查询调用.

### 4.4.5 限制查询结果

查询方法的结果可以通过关键`first`或者`top`限制,可以交换使用.一个可选的数值可以追加在top/first来指定返回的最大结果集.如果数字缺失,假定结果集大小是1.

例13.查询中使用Top和First限制结果大小

```java
User findFirstByOrderByLastnameAsc();
User findTopByOrderByAgeDesc();
Page<User> queryFirst10ByLastname(String lastname, Pageable pageable);
Slice<User> findTop3ByLastname(String lastname, Pageable pageable);
List<User> findFirst10ByLastname(String lastname, Sort sort);
List<User> findTop10ByLastname(String lastname, Pageable pageable);
```

限制表达式也支持Distinct关键字.此外，对于将结果集设置为一个实例的查询，支持将结果包装到一个`Optional`.如果应用分页或分片限制查询分页(并且计算可用的页数)那么这可以应用limited结果.

> 注意
>
> 请注意，限制结果结合使用`Sort`的动态排序的结果允许参数可以表示“k”最小的查询方法以及“K”的查询方法最大的元素。

### 4.4.6 流式处理结果

方法查询结果可以通过使用java 8`Stream<T>`逐步处理。

特定的方法用来表示流而不是简单的包装查询结果在一个`Stream`数据存储

例14 一个用java流8`Stream<T>`查询结果

```java
@Query("select u from User u")
Stream<User> findAllByCustomQueryAndStream();

Stream<User> readAllByFirstnameNotNull();

@Query("select u from User u")
Stream<User> streamAllPaged(Pageable pageable);
```

> 注意
>
> 一个`Stream`潜在的包装底层数据存储在页数的资源中,因此使用完毕必须关闭.你可以使用`close()`方法手动关闭`Stream`或者使用一个Java7的try_with-resources块.

```java
try(Stream<User> stream = repository.findAllByCustomQueryAndStream()) {
  stream.forEach(...);
}
```

### 4.4.7 异步查询结果

Repository查询可以使用`Spring's asynchronous method execution capacity`执行异步.这意味着方法可以一执行立即返回,真实的查询执行发生在任务提交到一个Spring TaskExecutor.

```java
@Async
Future<User> findByFirstname(String firstname);

@Async
CompletableFuture<User> findOneByFirstname(String firstname);

@Async
ListenableFuture<User> findOneByLastname(String lastname);
```

1. 使用`java.util.concurrent.Future`作为返回类型
2. 使用一个Java8`java.util.current.CompletableFuture`作为返回类型
3. 使用一个`org.springframework.util.concurrent.ListenableFuture`作为返回类型

## 创建repository实例

这个章节你创建实例和bean定义为repository接口定义.方法之一是手动的支持repository使用包含各个Spring Data模块的Spring命名空间装载,然而我们一般推荐使用Java配置的方法配置.

### 4.5.1 XML配置

每个Spring Data module包含一个repository 元素,这可以让你简单的定义一个基本包,Spring为你扫描它.

例16 通过XML启用Spring Data repositories

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <beans:beans xmlns:beans="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns="http://www.springframework.org/schema/data/jpa"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/data/jpa
        http://www.springframework.org/schema/data/jpa/spring-jpa.xsd">
      <repositories base-package="com.acme.repositories" />
    </beans:beans>
```

在前面的实例中,让Spring扫描`com.acme.repositories`包和它的子包里继承`Repository`的接口或者它的子接口.找到的每个接口,继承组件注册持久化技术的`FactoryBean`来创建合适的代理处理执行查询方法.每个bean被注册在一个接口名称确定的bean name下,所以一个叫`UserRepository`将注册`userRepository`.基本包属性允许通配符,所以你可以定义一个规则扫描包.

**使用过滤**

默认的基本组件选取所有在配置的基本包下继承了持久化技术`Repositpry`接口以及子接口

并且为他们创建一个bean实例.然而,你可能希望更细粒度的控制哪个接口bean实例被创建.为了实现这个你可以在`<repository/>`中使用`<include-filter/>`和`<exclude-filter/>`元素.语义完全等同于Spring的上下文命名空间中的元素.更多细节,查看他们的元素`Spring reference documentation`

例如，要将某些确定的接口排除实例化为repository，可以使用以下配置:

例17. 使用exclude-filter元素

```java
<repository base-package="com.acme.repositories">
  <context:exclude-filter type="regex" expression=".*SomeRepository" />
</repositories>
```

这个示例从待实例化对象中排除所有以SomeRepository结尾的接口.

### 4.5.2 JavaConfig

repository基础组件也可以使用一个存储的特殊的`@Enable${store}Repositories`注解在一个JavaConfig类上.入门介绍Spring容器Java基本Spring容器查看文档:`JavaConfig in the Spring reference documentaional`

一个简单配置启用Spring Data repositories像这样:

例18. repository配置基于简单注解

```java
@Configuration
@EnableJpaRepositories("com.acme.repositories")
class ApplicationConfiguration {
  @Bean
  public EntityManagerFactory entityManagerFactory() {
    // …
  }
}
```

> 注意
>
> 示例使用了JPA特有的注解,根据你使用的存储模块决定实际如何替换.示例定义`EntityManagerFactory`bean.查阅具体存储的配置

### 4.5.3 单独使用

你也可以在Spring容器之外使用repository基础组件,例如在CDI环境.你仍然需要一些Spring库在你的classpath中,但是你可以以编程的方式启动.repository包装持久化技术支持的Spring Data模块提供RepositoryFactory,你可以向下面这样使用:

例19. repository工厂单独使用

```java
RepositoryFactorySupport factory = … // Instantiate factory here
UserRepository repository = factory.getRepository(UserRepository.class);
```

## 4.6 定制Spring Data仓库实现

时常有必要为一少部分仓库方法提供一个定制的实现.Spring数据存储库很容易允许您提供自定义存储库代码并将其与通用CRUD集成抽象和查询方法功能整合.

### 4.6.1 为单独仓库添加定制行为

为了定制功能丰富一个仓库,你首先为定制功能定义一个接口和实现.使用你提供的仓库接口来继承自定义接口.

例20. 定制仓库功能的接口

```java
interface UserRepositoryCustom {
  public void someCustomMethod(User user);
}
```

例21.定制功能的实现

```java
class UserRepositoryImpl implements UserRepositoryCustom {
  public void someCustomMethod(User user) {
    // Your custom implementation
  }
}
```

> 注意
>
> 类可以被找到最重要的点是名字以Impl为后缀区别于仓库的核心接口(见下文)

实现的本身没有依赖Spring Data,可以是一个标准的Spring bean.所以你可以使用标准的依赖注入行为给其他bean注入引用,像`JdbcTemplate`,切面的一部分等等.

例22 修改你基本的仓库接口

```java
interface UserRepository extends CrudRepository<User, Long>, UserRepositoryCustom {
  // Declare query methods here
}
```

让你的标准仓库接口继承定制的.这样做结合了CRUD和定制功能并使其可用于客户端.

**配置**

如果你使用命名空间配置,仓库基本组件扫描类所在包,根据扫描结果尝试自动定制实现.

这些类需要遵循命名规范:给仓库接口名添加命名空间元素属性`repositoryimpl-postfix`.默认的后缀是`Impl`

例23. 配置示例

```xml
<repositories base-package="com.acme.repository" />
<repositories base-package="com.acme.repository" repository-impl-postfix="FooBar"/>
```

第一个配置示例将查实查找一个类`com.acme.repository.UserRepositoryImpl`来作为定制藏局实现.然而第二个示例将尝试查找`com.acme.repository.UserRepositoryFooBar `.


**手动指定**

上面的方法可以正常工作,只有当你定制实现使用注解配置和自动注入,这将与其他Spring bean一样被对待.如果你定制的实现需要特殊处理,你可以像描述的简单定义一个bean并且命名它.基本组件将通过名称引用手动定义的bean定义而不是它自己创建一个.

例24.手动指定定制实现

```xml
<repositories base-package="com.acme.repository" />
<beans:bean id="userRepositoryImpl" class="…">
  <!-- further configuration -->
</beans:bean>
```

### 4.6.2为所有仓库添加定制行为

当你希望把一个单独的方法添加到你所有的仓库接口中时,上面的方法就不可行了.为了添加定制到所有的仓库,你首先添加一个中间接口来定义共享的行为.

例25 定义共享定制行为接口

```java
@NoRepositoryBean
public interface MyRepository<T, ID extends Serializable>
  extends PagingAndSortingRepository<T, ID> {
  void sharedCustomMethod(ID id);
}
```

现在你独立的仓库接口将继承这个中间接口而不是`Repository`接口来包含功能的定义.接下来创建一个中间接口的实现继承持久化具体仓库的基本类.这个类后面将作为仓库代理的基本类.

例26 定制仓库基本类

```java
public class MyRepositoryImpl<T, ID extends Serializable>
  extends SimpleJpaRepository<T, ID> implements MyRepository<T, ID> {
  
  private final EntityManager entityManager;
  
  public MyRepositoryImpl(JpaEntityInformation entityInformation,
                          EntityManager entityManager) {
    super(entityInformation, entityManager);
    // Keep the EntityManager around to used from the newly introduced methods.
    this.entityManager = entityManager;
  }
  
  public void sharedCustomMethod(ID id) {
    // implementation goes here
  }
}
```

> 警告
>
> 这个类需要有一个构造方法调用父类具体存储仓库工厂实现.万一仓库基础类有多个构造,覆盖包括一个`EntityInformation `加上存储具体基本组件对象(例如一个`EntityManager `或者模板类)

Spring`<repository/>`命名空间下的默认行为为所有接口提供一个实现.这意味着如果处于其当前状态，`MyRepository`的实现实例将由Spring创建.这当然不是被期望的,它只是作为一个用来定义实体的 `Repository`和真实仓库接口的中间接口.为了排除一个继承`Repository`的接口被当做一个仓库接口实例化,你可以给它使用`@NoRepositoryBean`(像上面)或者把它从配置中`base-package`移除.

最后一步是让Spring Data基本组件识别到定制的仓库基本类.在JavaConf使用注解`@Enable...Repository`的属性`repositoryBaseClass`完成:

例27 使用JavaConfig配置一个定制仓库基本类

```java
@Configuration
@EnableJpaRepositories(repositoryBaseClass = MyRepositoryImpl.class)
class ApplicationConfiguration { … }
```

类似的属性在XML命名空间中也可以找到.

例28 使用XML配置一个定制仓库基本类

```xml
<repositories base-package="com.acme.repository"
              base-class="….MyRepositoryImpl" />
```

## 4.7 Spring Data扩展

这部分说明Spring Data一系列的扩展功能,可以使Spring Dta使用多样的上下文.目前大部分集成是针对Spring MVC.

### 4.7.1 Querydsl扩展

Querydsl是一个框架,通过它的流式API构建静态类型的SQL类查询.

多个Spring Data模块通过`QueryDslPredicateExecutor `与Querydsl集成.

例29 QueryDslPredicateExecutor接口

```java
public interface QueryDslPredicateExecutor<T> {
  T findOne(Predicate predicate); ①
    Iterable<T> findAll(Predicate predicate); ②
    long count(Predicate predicate); ③
    boolean exists(Predicate predicate); ④
    // … more functionality omitted.
}
```

① 查询并返回一个匹配`Predicate `的单例实体

②查询并返回所有匹配`Predicate`的实体

③ 返回匹配`Predicate`的实体数量

④  返回是否存在一个匹配`Predicate`的实体

为了简单的使用Querydsl功能,在你的仓库接口继承`QueryDslPredicateExecutor`.

例30 在仓库集成QueryDsl

```java
interface UserRepository extends CrudRepository<User, Long>,
  QueryDslPredicateExecutor<User> {
}
```

像上面这样就可以使用Querydsl的`Predicate`书写类型安全的查询

```java
Predicate predicate = user.firstname.equalsIgnoreCase("dave")
  .and(user.lastname.startsWithIgnoreCase("mathews"));
userRepository.findAll(predicate);
```

### 4.7.2 Web支持

> 注意
>
> 本节包含Spring Data Web支持的文档是在1.6范围内的Spring Data Commons实现的.因为支持新引入的内容改变了很多东西，我们保留了旧行为的文档在"遗留Web支持"部分.
>

如果模块支持仓库编程模型，那么Spring Data模块附带了各种Web模块支持.Web关联的东西需要Spring MVC的JAR包位于classpath路径下,它们中有些甚至提供了Spring HATEOAS集成.一般情况,集成方式支持使用`@EnableSpringDataWebSupport`注解在你的JavaConfig配置类.

例31 启用Spring Data web支持

```java
@Configuration
@EnableWebMvc
@EnableSpringDataWebSupport
class WebConfiguration {}
```

`@EnableSpringDataWebSupport`注解注册了一些组件，我们将在稍后讨论.注解还将在类路径上检测Spring HATEOAS，如果才在将为其注册集成组件.

作为可选项,如果你使用XML配置,注册`SpringDataWebSupport`或者`HateoasWareSpringDataWebSupport`作为Spring bean:

例32 用XML启用Spring Data web支持

```xml
<bean class="org.springframework.data.web.config.SpringDataWebConfiguration" />
<!-- If you're using Spring HATEOAS as well register this one *instead* of the
former -->
<bean class= "org.springframework.data.web.config.HateoasAwareSpringDataWebConfiguration" />
```

**基本Web支持**

上面展示的的配置设置将注册几个基本组件：

- 一个`DomainClassConverter`启用Spring MVC来根据请求参数或路径变量管理仓例实体类的实例
- `HandlerMethodArgumentResolver`实现让Spring MVC从请求参数解析Pageable和Sort实例.

**实体类转换**

`DomainClassConverter`允许你在Spring MVC控制器方法签名中直接使用实体类型,因此你不必手动的通过仓库查询实例:

例33 一个Spring MVC控制器在方法签名中使用实体类型

```java
@Controller
@RequestMapping("/users")
public class UserController {
  @RequestMapping("/{id}")
  public String showUserForm(@PathVariable("id") User user, Model model) {
    model.addAttribute("user", user);
    return "userForm";
  }
}
```

如你所见,方法直接接收一个User实例并没有更进一步的查询是否必要.实例可以通过Spring MVC将路径变量转换为实体类的id类型并最终通过在实体类型注册的仓库实例上调用`findOne(...)`访问实例转换得到.

> 注意
>
> 当前的仓库必须实现`CrudRepository`做好准备被发现来进行转换.

**为了分页和排序分解方法参数**

上面的配置片段还注册了一个`PageableHandlerMethodArgumentResolver`和一个`SortHandlerMethodArgumentResolver`实例.注册使得Pageable和Sort成为有效的控制器方法参数.

例34 使用Pageable作为控制器方法参数

```java
@Controller
@RequestMapping("/users")
public class UserController {
  @Autowired UserRepository repository;
  @RequestMapping
  public String showUsers(Model model, Pageable pageable) {
    model.addAttribute("users", repository.findAll(pageable));
    return "users";
  }
}
```

这个方法签名将使Spring MVC尝试使用下面的默认配置从请求参数中转换一个Pageable实例:

表1 请求参数转换Pageable实例

| 参数   | 说明                                       |
| ---- | ---------------------------------------- |
| page | 要检索的页面,索引为0,默认为0                         |
| size | 要检索的页面大小,默认20                            |
| sort | 被排序的属性应以格式`property,property(, ASC\|DESC)`表示.默认生序排序,如果你希望改变排序顺序,则使用多个排序参数,例如?sort=firstname&sort=lastname,asc |

为了定制行为,可以继承`SpringDataWebConfiguration`或者启用等效的HATEOAS并覆盖`pageableResolver()`或`sortResolver()`方法并导入你的自定义配置文件替代@Enable-注解.

有一种情况你需要多个`Pageable`或`Sort`实例从请求转换(例如处理多个表单),你可以使用Spring的`@Qualifier`注解来互相区别.请求参数必须以`${qualifier}为`前缀.这样一个方法的签名像这样:

```java
public String showUsers(Model model, 
                        @Qualifier("foo")Pagebale first, 
                        @Qualifier("bar") Pageable second) {
  ...
}
```

你必须填充foo_page和bar_page等.

默认的`Pageable`在方法中处理等价于一个`new PageRequest(0, 20)`,但是可以使用`@PageableDefaults`注解在`Pageable`参数上定制.

**Hypermedia支持分页**

Spring HATEOAS包装了一个代表模型的类`PageResources` ,

它可以使用Page实例包装必要的Page元数据内容作为连接让客户端导航页面.一个页面到一个`PageResources`的转换被Spring HATEOAS的`ResourceAssembler`接口实现`PagedResourcesAssembler `来完成.

例35 使用一个PagedResourcesAssembler作为控制器方法参数

```java
@Controller
class PersonController {
  @Autowired PersonRepository repository;
  @RequestMapping(value = "/persons", method = RequestMethod.GET)
  HttpEntity<PagedResources<Person>> persons(Pageable pageable,
                                             PagedResourcesAssembler assembler) {
    Page<Person> persons = repository.findAll(pageable);
    return new ResponseEntity<>(assembler.toResources(persons), HttpStatus.OK);
  }
}
```

像上面这样配置将允许`PageResourcesAssembler`作为控制器方法的一个参数.在这调用toResources(...)方法有以下作用:

- `Page`的内容将`PageResources`实例的内容
- `PageResources`将获得`PageMetadata`实例,该实例由Page和基础的PageRequest中的信息填充
- `PageResources`获得`prev`和`next`连接,添加这些依赖在页面.这些链接将指向uri方法的调用映射.页码参数根据`PageableHandlerMethodArgumentResolver`添加到参数以在后面被转换.

假设我们有30个Person实例在数据库.你现在可以触发一个GET请求 http://localhost:8080/persons, 你将可以看到类似下面的内容:

```json
{ "links" : [ { "rel" : "next",
"href" : "http://localhost:8080/persons?page=1&size=20 }
],
"content" : [
… // 20 Person instances rendered here
],
"pageMetadata" : {
"size" : 20,
"totalElements" : 30,
"totalPages" : 2,
"number" : 0
}
}
```

你可以看到编译生成了正确的URI，并且还会提取默认配置转换参数到即将到来的请求中的`Pageable`.这意味着,如果你改变配置,链接也将自动跟随改变.默认情况下，编译指向控制器执行的方法，但是这可以被一个自定义链接作为基本构建来构成分页的`Link`重载`PagedResourcesAssembler.toResource（...）`方法定制.

**Querydsl web 支持**

那些整合了`QueryDSL`的存储可能从`Request`查询字符串中的属性驱动查询.

这意味着前面例子的查询字符串可以给出`User`的对象

```
?firstname=Dave&lastname=Matthews
```

可以被转换为

```java
QUser.user.firstname.eq("Dave").and(QUser.user.lastname.eq("Matthews"))
```

使用了`QuerydslPredicateArgumentResolver`.

> 注意
>
> 当在类路径上找到Querydsl时，该功能将在@EnableSpringDataWebSupport注解中自动启用

添加一个`@QuerydslPredicate`到一个方法签名将提供一个就绪的`Predicate`,可以通过`QueryDslPredicateExecutor`执行.

> 提示
>
> 类型信息通常从返回方法上解析.由于这些信息不一定匹配实体类型,使用`QuerydslPredicate`的`root`属性可能是个好主意.

```java
@Controller
class UserController {
  @Autowired UserRepository repository;
  @RequestMapping(value = "/", method = RequestMethod.GET)
  String index(Model model, @QuerydslPredicate(root = User.class) Predicate predicate,
              Pageable pageable, @RequestParam MultiValueMap<String, String>
    parameters) {
        model.addAttribute("users", repository.findAll(predicate, pageable));
        return "index";
  } 
}
```

为User转换匹配查询字符串参数的`Predicate`

默认的绑定规则如下:

1. `Object`在简单属性上如同`eq`

2. `Object`在集合作为属性如同`contains`

3. `Collection`在简单属性上如同`in`


这些绑定可以通过`@QuerydslPredicate`的`bindings`属性定制或者使用Java8`default methods`给仓库接口添加`QuerydslBinderCustomizer`

```java
interface UserReposotory extends CurdRepository<User, String>, 
  QueryDslPredicateExecutor<User>,
  QuerydslBinderCustomizer<QUser> {
    @Override
  	default public void customize(QuerydslBindings bindings, QUser user) {
      bindings.bind(user.username).first((path, value) -> path.contains(value));
      bindings.bind(String.class).first((StringPath path, String value) ->
                                        path.containsIgnoreCase(value));
      bindings.excluding(user.password);
  	}
  }
```

1. `QueryDslPredicateExecutor`为`Predicate`提供特殊的查询方法提供入口
2. 在仓库接口定义`QuerydslBinderCustomizer`将自动注解`@QuerydslPredicate(bindings=...)`
3. 为`username`属性定义绑定,绑定到一个简单集合
4. 为`String`属性定义默认绑定到一个不区分大小写的集合
5. 从`Predicate`移除密码属性

### 4.7.3 仓库填充

如果你使用Spring JDBC模块,你可能熟悉在`DataSource`使用SQL脚本来填充.一个类似的抽象在仓库级别可以使用,尽管它不是使用SQL作为数据定义语言,因为它必须由存储决定.填充根据仓库支持XML(通过Spring的OXM抽象)和JSON(通过Jackson)定义数据.

假设你有一个文件`data.json`内容如下:

例36 JSON定义的数据

```json
[ { "_class" : "com.acme.Person",
     "firstname" : "Dave",
      "lastname" : "Matthews" },
      { "_class" : "com.acme.Person",
     "firstname" : "Carter",
      "lastname" : "Beauford" } ]
```

你可以容易的根据Spring Data Commons提供仓库的命名空间填充元素填充你的仓库.为了填充前面的数据到你的PersonRepository,像下面这样配置:

例37 声明一个Jackson仓库填充

```xml
  <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:repository="http://www.springframework.org/schema/data/repository"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/data/repository
        http://www.springframework.org/schema/data/repository/spring-repository.xsd">
      <repository:jackson2-populator locations="classpath:data.json" />
    </beans>
```

这样的声明可以让`data.json`文件可以被一个Jackson的`ObjectMpper`读取和反序列化.

JSON将要解析的对象类型由检查JSON文档的`_class`属性决定.基本组件将最终选择合适的仓库去处理反序列化的对象.

要使用XML定义数据填充仓库,你可以使用`unmarshaller-populator`元素.你配置它使用Spring OXM提供给你的XML装配选项.在Spring reference documentation查看更多细节.

例38 声明一个装配仓库填充器(使用JAXB)

```xml
<?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:repository="http://www.springframework.org/schema/data/repository"
      xmlns:oxm="http://www.springframework.org/schema/oxm"
      xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/data/repository
        http://www.springframework.org/schema/data/repository/spring-repository.xsd
        http://www.springframework.org/schema/oxm
        http://www.springframework.org/schema/oxm/spring-oxm.xsd">
      <repository:unmarshaller-populator locations="classpath:data.json"
        unmarshaller-ref="unmarshaller" />
      <oxm:jaxb2-marshaller contextPath="com.acme" />
    </beans>
```

### 4.7.4 遗留web支持

**Spring MVC的实体类绑定**

如果正在开发Spring MVC web应用,你通常必须从URL中解析实体类的id.默认的,你的任务是转化请求参数或URL参数到实体类并将它移交给下面或直接在实体上操作业务逻辑.这看起来像下面这样:

```java
@Controller
@RequestMapping("/users")
public class UserController {
  private final UserRepository userRepository;
  
  @Autowired
  public UserController(UserRepository userRepository) {
    Assert.notNull(repository, "Repository must not be null!");
    this.userRepository = userRepository;
  }

  @RequestMapping("/{id}")
  public String showUserForm(@PathVariable("id") Long id, Model model) {
    // Do null check for id
    User user = userRepository.findOne(id);
    // Do null check for user
    model.addAttribute("user", user);
    return "user";
  }
}
```

首先你为每个控制器定义一个依赖的仓库来查找它们分别管理的实体.查询实体也是样板,因为它总是一个`findOne(...)`调用.幸运的Spring提供了方法来注册自定义组件,允许一个String值转换到一个属性类型.

**属性编辑**

Spring3.0之前Java`PropertyEditors`被使用.为了集成这些,Spring Data提出一个`DomainClassPropertyEditorRegistrar`来查询所有注册到`ApplicatonContext`的Spring Data仓库和一个定制的`PropertyEditor`来管理实体类.

```xml
<bean class="….web.servlet.mvc.annotation.AnnotationMethodHandlerAdapter">
  <property name="webBindingInitializer">
    <bean class="….web.bind.support.ConfigurableWebBindingInitializer">
      <property name="propertyEditorRegistrars">
        <bean class=
          "org.springframework.data.repository.support.DomainClassPropertyEditorRegistrar" />
      </property>
    </bean>
  </property>
</bean>
```

如果你已经像上面这样配置Spring MVC,你可以向下面这样配置你的控制器,从而减少不清晰和样板式的代码

```java
@Controller
@RequestMapping("/users")
public class UserController {
  @RequestMapping("/{id}")
  public String showUserForm(@PathVariable("id") User user, Model model) {
    model.addAttribute("user", user);
    return "userForm";
  }
}
```

