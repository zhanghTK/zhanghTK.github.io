---
title: JAVA碎碎念【一】
date: 2018-03-04 20:10:21
tags: Java
categories: Java
---

过去的 2017 年，关于 Java 大大小小的坑还是踩了不少。这里回顾一下还记得的问题。

**数组与List的转换**

常见做法：

`List<String> list = Arrays.asList(arr);`

问题：

多数我们都认为`Arrays.asList(arr)`方法返回的是ArrayList.

实际上返回的是`java.util.Arrays.ArrayList`。

声明ArrayList声明：

`private static class ArrayList<E> extends AbstractList<E> implements RandomAccess, Serializable`

实现方法如下图：

![java.util.Arrays.ArrayList结构.jpg](https://ooo.0o0.ooo/2017/01/05/586dfe79029b6.jpg)

`java.util.Arrays.ArrayList`并没有实现所有`ArrayList`的方法

所以当我们期望得到一个真正的`ArrayList`时需要如下操作：

`ArrayList<String> arrayList = new ArrayList<String>(Arrays.asList(arr));`

类似的问题：
`java.util.ArrayList.subList()` 返回的不是 ArrayList。当在有些场景下需要依赖容器特性（例如是否序列化）时要确认一下集合操作的返回类型，如有必要最好显示的转换。

---

**集合中移除元素**

例如，循环删除List中所有元素，很容易写出下面的代码：

```java
List<String> list = new ArrayList<String>(Arrays.asList("a", "b", "c", "d"));
for (int i = 0; i < list.size(); i++) {
  list.remove(i);
}
```

问题：

每次根据索引删除元素后List的大小及元素索引会发生变化，需要使用迭代器实现，使用迭代器很容易写出下面的代码：

```java
List<String> list = new ArrayList<String>(Arrays.asList("a", "b", "c", "d"));
for (String s : list) {
  if (s.equals("a"))
    list.remove(s);
}
```

实际还是错误的，正确写法如下：

```java
List<String> list = new ArrayList<String>(Arrays.asList("a", "b", "c", "d"));
Iterator<String> iter = list.iterator();
while (iter.hasNext()) {
  String s = iter.next();
 
  if (s.equals("a")) {
    iter.remove();
  }
}
```

---

**`Integer.parseInt(String s)/valueOf(String s)`**

`Integer.parseInt(String s) `和 `Integer.valueOf(String s)` 的区别

将一个数字字符串转数字，例如：

```java
// “123” -> 123
String numStr = "123";
Integer.parseInt(numStr);
Integer.valueOf(numStr);
```

`Integer.parseInt()` 和 `Integer.valueOf()` 的区别主要在于放回的类型上：

`Integer.parseInt()` 返回 `int`

`Integer.valueOf()`返回`Integer`

对性能敏感的地方或循环内部，应当避免不必要的拆箱装箱

---

`ConcurrentMap.putIfAbsent(key,value) `

错误用法：

```java
ConcurrentMap<Dept, Person> cash = new ConcurrentMap.putIfAbsent<>();
// ...
public Dept getDept(Person person) {
  Dept dept = map.get(person);
  if (dept == null) {
    dept = person.getDept();
    map.putIfAbsent(person, dept);
  }
  return dept;
}
```

`putIfAbsent`方法完整签名如下：`public V putIfAbsent(K var1, V var2) `

看到返回值了有木有，这点很关键。

多线程环境下，可能线程A首先获取失败，当A创建完value对象，准备插入时已经有别的线程插入了，这时候再返回A创建的value可能存在问题。

所以，如果当前put失败，则返回已存在的value，否则返回null。

正确的写法：

```java
ConcurrentMap<Dept, Person> cache = new ConcurrentMap.putIfAbsent<>();
// ...
public Dept getDept(Person person) {
  Dept dept = cachecache.get(person);
  if (dept == null) {
    dept = person.getDept();
    Dept deptPuted = cache.putIfAbsent(person, dept);
    if (deptPuted != null) {
      dept = deptPutted;
    }
  }
  return dept;
}
```
---

**过长的参数**

如果一个方法有一个长长的参数，首先考虑的**不是**通过把多个参数封装在一起。

如果一个方法需要多个参数不能从自己的宿主类获得，那么这个方法的位置可能有问题，要警惕这点。

---

**双胞胎`Class.forName`**

Class.forName：返回与给定的字符串名称相关联类或接口的Class对象

该方法有两种形式：

`Class.forName(String name, boolean initialize, ClassLoader loader)`

- name表示的是类的全名
- initialize表示是否初始化类
- loader表示加载时使用的类加载器

` Class.forName(String className)`

等价于：`Class.forName(name, true, this.getClassLoader)`

---

**链接的有效性测试**

Web开发中页面与控制器的映射通常是使用配置完成。

静态语言一大优势是编译期的检查，帮助我们重构代码，发现错误。但无法检查配置是否正确，例如：

在维护Struct1项目过程中我们有一个基本的抽象Action`ExtendAction`提供了页面CRUD访问的入口，多数的Action都继承`ExtendAction` ，现在有个报表的抽象类`AbstractReportAction`也是继承了`ExtendAction`。

某次修改希望为报表Action增加新功能，目前看到需求报表都没有使用到CRUD功能，因此扩展的方式是通过重构，引入了新的抽象Action代替了`AbstractReportAction`的父类`ExtendAction`

另一个需求需要使用到CRUD的功能，因为继承的`AbstractReportAction`是继承自`ExtendAction`的可以直接使用。

现在两个需求开发完成，分别在自己分支上测试完成，合并代码后，编译正确。

如果合并后，结果依赖分支测试，没有充分集成测试，上线后可能就会造成事故。

改进：

1. 少用继承，这次的案例中一定程度是因为多层次的继承关系隐蔽了错误；
2. 对基础类的修改，需要谨慎，团队的所有成员需要对此保持警惕，细心评估可能现阶段及未来可能带来的风险。
3. 对链接的有效性测试，在单元测试层面添加测试是非常有必要的。
4. 除了单元测试，当多个功能集成后，集成测试也很关键。

---

**单元测试框架补充——Junit**

平时对JUnit使用比较简单，今天看到一个介绍Junit的：

- JUnit中的Assert方法：

  - `void assertEquals(boolean expected, boolean actual)`检查两个变量或者等式是否平衡

  - `void assertFalse(boolean condition)`检查条件是假的

  - `void assertNotNull(Object object)`检查对象不是空的

  - `void assertNull(Object object)`检查对象是空的

  - `void assertTrue(boolean condition)`检查条件为真

  - `void fail()`在没有报告的情况下使测试不通过

- JUnit中的注解

  - `@BeforeClass`：针对所有测试，只执行一次，且必须为static void
  - `@Before`：初始化方法
  - `@Test`：测试方法，在这里可以测试期望异常和超时时间
  - `@After`：释放资源
  - `@AfterClass`：针对所有测试，只执行一次，且必须为static void
  - `@Ignore`：忽略的测试方法

- 时间测试

  ```java
  @Test(timeout = 1000)
  public void testTimeoutSuccess() {
      // do nothing
  }
  ```

- 异常测试

  ```java
  @Test(expected = NullPointerException.class)
  public void testException() {
      throw new NullPointerException();
  }
  ```

---

**反射中的`isAccessible`**

isAccessible()返回值的含义：

true：直接访问，不受检查

false：不能直接访问，接受检查

所有的accessible标志默认都为false，哪怕它使public的。

setAccessible设置是否开启直接访问（关闭安全检查），反射调用提升速度很关键。

这里很容易误解成，private 的成员调用 isAccessible 返回 true，public 的成员调用 isAccessible 返回 false。 

举个例子：

```java
public class Employee {
    private String id;
    public String name;
    // 省略构造，toString()
}

public class AccessibleTest {
    public static void main(String[] args) throws IllegalAccessException, NoSuchFieldException {
        Employee employee = new Employee("1", "ZS");
        System.out.println(employee);
        Class<?> clazz = employee.getClass();
        printFieldAccess(clazz, "id");
        // id isPublic:false    id isAccessible:false
        printFieldAccess(clazz, "name");
		// name isPublic:true    name isAccessible:false
        changeFieldValue(employee, "name");
        // Employee{id='1', name='TEST'}
        changeFieldValue(employee, "id");
        // java.lang.IllegalAccessException
    }

    private static void printFieldAccess(Class<?> clazz, String fieldName) throws NoSuchFieldException {
        Field field = clazz.getDeclaredField(fieldName);
        System.out.println(field.getName() + " isPublic:" + Modifier.isPublic(field.getModifiers()));
        System.out.println(field.getName() + " isAccessible:" + field.isAccessible() + "\n");
    }

    private static void changeFieldValue(Object object, String fieldName) throws NoSuchFieldException, IllegalAccessException {
        Class clazz = object.getClass();
        Field field = clazz.getDeclaredField(fieldName);
        field.set(object, "TEST");
        System.out.println(object);
    }
}
```

---

**反射API中的兄弟方法**

反射API中有几对兄弟，可以概括为`getXxx()`/`getDeclaredXxx()`

`getXxx()` ：获取所有的公有Xxx，包括继承来的

`getDeclaredXxx`：获取“当前类“的所有Xxx，不受访问权限限制

---

**基本/包装类型在反射中坑**

JDK5之后基本类型与包装类型之间转换已经可以做到自动拆箱/装箱了，日常开发不再留意这些，但是反射中还是要自己注意类型，例如：

```java
public class ReflectBoxingTest {
    private static class Employee {
        public void testBaeType(int x) {
        }

        public void testBoxingType(Integer x) {
        }
    }

    public static void main(String[] args) {
        Class<Employee> clazz = Employee.class;
        getMethod(clazz, "testBaeType", int.class);
        getMethod(clazz, "testBaeType", Integer.class);
        getMethod(clazz, "testBoxingType", int.class);
        getMethod(clazz, "testBoxingType", Integer.class);
    }

    private static void getMethod(Class<?> clazz, String method, Class<?>...classes) {
        try {
            clazz.getDeclaredMethod(method, classes);
        } catch (NoSuchMethodException e) {
            System.out.println("can't get method " + method 
                               + "(" + Arrays.toString(classes) + ")");
        }
    }
}
```

很容易让人觉得上述四种方式均能过去到method，实际上`int.class`与`Integer.class`之间是不能自动转型的。

输出结果：

```
can't get method testBaeType([class java.lang.Integer])
can't get method testBoxingType([int])
```

引申：在ORM框架中的问题等等。

---

**谨慎使用DB中`not-null="false"`**

如果对一个字段设置了`not-null=true`,那么在以后的使用当中无论是发送SQL语句还是处理实体类，都必须先对其作出判空处理。例子：

新添加了一个字段期望管理员限定是否只在移动端访问

Hibernate配置文件中：

```xml
<property
          name="fdMobileOnly"
          column="fd_mobile_only"
          update="true"
          insert="true"
          not-null="false"
          length="1" />
```

HQL语句：

```java
where = ....
where += " and entity.fdMobileOnly != 1";
```

代码发布后，管理员没有第一时间更新所有字段，用户访问后出错，当然这不是管理员的错。这样的错误在开发阶段是可以避免的，测试阶段，发布阶段都可以做些什么，但是我们缺少相应的支持。

---

**内省**

比反射更便捷的操作 bean 实例，使用内省机制。[内省Demo](https://github.com/zhanghTK/JavaCodeRepository/blob/master/practice/src/main/java/tk/zhangh/java/practice/IntrospectorTest.java)

---

**异常处理**

尽可能的简单，复杂的异常处理过程本身就可能包含了新的异常，在异常处理中发生新的异常可能就会覆盖原始异常，这不是我们想看到的。

---

**Thread.run()/start()**

start()：用于启动线程

run()：普通方法

直接调用run方法将不会开启新的线程，依旧是顺序执行。

---

**小数问题**

`System.out.println(0.1f+0.1f);  // 0.2`

`System.out.println(0.1f*0.1f);  // 0.010000001`

第一行结果接近0.2所以输出时系统选择输出了0.2

---

**泛型问题**

Java实现泛型是使用泛型擦除的

Java的泛型不能协变

可以使用通配符指定不确定的泛型

* 生产者使用extends
  * <? extends T> 声明的类型是T的子类，具体哪个子类不知道，所以不能随便给？赋引用，因为可能出现转型错误，但是知道是T的子类，肯定实现了T的方法
* 消费者使用super
  * <? super T>   声明的类型是T的父类，具体哪个父类不知道，反正至少T类型，所以T的子类型都可以被？引用，但引用的是什么类型不知道，所以根据？获取的都是object

---

**线程创建**

在Java中创建线程，一个线程默认就会预留1M的空间，那么1G的内存也不过只能支持1000个线程创建而已

---

**获取泛型类型T的类实例**

```java

import org.springframework.core.GenericTypeResolver;
public abstract class AbstractHibernateDao<T extends DomainObject> implements DataAccessObject<T>
{
@Autowired
private SessionFactory sessionFactory;
private final Class<T> genericType;
private final String RECORD_COUNT_HQL;
private final String FIND_ALL_HQL;
@SuppressWarnings("unchecked")
public AbstractHibernateDao()
{
 this.genericType = (Class<T>) GenericTypeResolver.resolveTypeArgument(getClass(), AbstractHibernateDao.class);
 this.RECORD_COUNT_HQL ="select count(*) from" + this.genericType.getName();
 this.FIND_ALL_HQL ="from" + this.genericType.getName() +" t";
}

```
---

**ThreadPoolExecutor.submit()/execute()**

ThreadPoolExecutor 线程池中 submit 和 execute 方法对比

1. submit 提交的任务可以获取执行结果，而execute 则不能
2. execute 会抛出异常，而submit 如果不获取执行结果的话会“吞掉”异常

详细分析：http://blog.techbeta.me/2016/08/ThreadPoolExecutor/

---

**ThreadPoolExecutor 线程池对异常处理**

1. 捕获 execute 抛出异常
2. submit 执行完成必须 get 一下检测异常
3. 扩展线程池，实现 beforeExecute 方法

---

**转型问题**

即使是同一个类文件，如果是由不同的类加载器实例加载的，那么它们的类型是不相同的。例如：

```java
public void run(){ 
    try { 
        // 每次都创建出一个新的类加载器
        HowswapCL cl = new HowswapCL("../swap", new String[]{"Foo"}); 
        Class cls = cl.loadClass("Foo"); 
        Object foo = cls.newInstance(); 
 
        Method m = foo.getClass().getMethod("sayHello", new Class[]{}); 
        m.invoke(foo, new Object[]{}); 
     
    }  catch(Exception ex) { 
        ex.printStackTrace(); 
    } 
}
```

cls 是由 HowswapCL 加载的，而 foo 变量类型声名和转型里的 Foo 类却是由 run 方法所属的类的加载器（默认为 AppClassLoader）加载的，因此是完全不同的类型，所以会抛出转型异常

解决：
http://www.wisedream.net/2017/01/17/programming/type-cast-across-classloader/
