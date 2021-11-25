[TOC]

# 周瑜 - Spring底层核心原理解析

- 本节课主要对 Spring 核心知识串讲，详细深入的在后续

## 示例代码

- 简单示例，Spring基础不了解的，可以查阅本文同级目录下的
  - 徐庶 - Spring核心相关概念

```java
public class Test {
  
  public static void main(String[] args) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
    UserService userService = (UserService) context.getBean("userService");
    userService.test();
  }
  
}
```

## Bean的创建生命周期底层原理

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1635213100449.png)

1. UserService.class 
2. -> 构造方法（推断构造方法）
3. -> 普通对象（实例化）
4. -> 依赖注入（属性赋值）
5. -> 初始化前（@PostConstruct）
6. -> 初始化（InitializingBean）
7. -> 初始化后（AOP, 决定是否为代理对象）
8. -> Bean

### 依赖注入底层原理

- Spring 创建 Bean，实例化后，进行属性赋值（依赖注入）的大概原理伪代码如下：

```java
for (Field field : userService.getClass().getFields()) {
  if (field.isAnnotationPresent(Autowired.class)) {
    // 就是获取 bean 所有属性，判断是否有 @Autowired 注解（可能还有 @Resource 注解之类等情况）
    // 有的话就做依赖注入操作（其实就是看 beanFactory 里有没有该 属性bean，没有就创建该 属性bean 再赋值）
    // 循环依赖的问题这里先不讨论
    field.set(userService, ??);
  }
}
```

### 初始化底层原理

#### 初始化前

- 属性赋值完后，到 初始化前，大概原理伪代码如下：

```java
for (Method method : userService.getClass().getMethods()) {
  if (method.isAnnotationPresent(PostConstruct.class)) {
    // 获取 bean 所有方法，判断是否有标记注解 @PostConstruct 的
    // 有的话就执行该方法
    method.invoke(userService, null);
  }
}
```

#### 初始化

- 大概原理伪代码如下

```java
if (userService instanceof InitializingBean) {
  // 判断 bean 是否实现了 InitializingBean 类
  // 实现了就强转为该类型，执行起 afterPropertiesSet() 方法
  // 见名思议，就是 属性赋值 之后执行的方法
  ((InitializingBean) bean).afterPropertiesSet();
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1635213905476.png)

## 推断构造方法底层原理

- 推断构造方法：
  - Spring 创建 bean 时，确定用 bean 的哪个构造方法，确定入参的 bean 对象
- 三种情况：
  1. 类只有一个构造方法，直接使用
  2. 类有多个构造方法
     1. 有无参构造，没有指定 @Autowired 的构造，使用无参构造
     2. 有无参构造，指定 @Autowired 的构造，使用 @Autowired 构造
     3. 无无参构造，没有指定 @Autowired 构造，报错抛异常
- Spring使用有参构造时，参数的获取：
  1. 根据 入参bean类型、或 入参bean名称，去单例池中找
  2. 先根据入参类型，只知道一个，则直接用来入参
  3. 根据类型找到多个时，根据入参名字来确定使用哪个
     1. 根据入参名字还找到多个就报错
  4. 没找到 入参bean 也报错

## AOP底层原理

### 简介

- AOP就是进行动态代理
  - 在创建一个 bean 的过程中，Spring会在最后一步判断
  - 当前bean是否需要进行AOP动态代理，如果需要，则创建代理类

### 如何判断bean是否需要代理

1. 找出所有切面bean
2. 遍历切面中的每个方法，看是否有 @Before、@After 等注解
3. 如果写了，判断所对应的 pointcut 是否和 当前bean 匹配
4. 如果匹配，则表示需要创建该 bean 的动态代理对象

### Cglib进行代理增强的大致流程

1. 生成代理类 XxxServiceProxy，代理类继承 XxxService
2. 代理类中重写父类方法，比如 test()
3. 代理类中有一个 `target` 属性，target = 目标对象（这里的 XxxService）
   1. 目标对象，是经过正常 bean 创建流程的对象
   2. 即: 完成了依赖注入（属性bean有值）、初始化等操作..
4. 代理类中的 test() 方法被执行时，
   1. 执行切面增强逻辑，比如: @Before
   2. 调用 target.test()

### 注意

- 从 Spring 中获取 XxxService 时，实际得到的是 XxxServiceProxy 代理对象

## Spring事务底层原理

### 简介

- 某个方法上加了 `@Transactional` 注解后，
  - 表示该方法会在调用后，开启 Spring事务
  - 该方法所在的 bean 也变成了 动态代理对象

### 流程

1. 判断当前执行方法，是否存在 @Transactional 注解
2. 如果存在，则利用 TransactionManager 新建一个 数据库连接
3. 修改数据库连接的 autocommit 为 false
4. 执行目标方法
5. 成功则提交、异常则回滚

### 事务是否失效的判断标准

- @Transactional 方法被调用时，
  - 判断是否为 `代理对象` 调用的
    - 是：事务生效
    - 否：事务失效
- 是否同一 datasource