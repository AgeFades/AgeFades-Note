[TOC]

# 徐庶 - Spring核心节点 & IOC加载

[Spring中文文档](https://github.com/DocsHome/spring-docs/blob/master/SUMMARY.md)

[Spring IOC加载流程图](https://www.processon.com/view/5e0a0bb7e4b0250e8afe4230?fromnew=1)

[Spring Bean生命周期](https://www.processon.com/view/5f087b1e7d9c087fac03f7b1?fromnew=1)

[Spring源码脑图](https://www.processon.com/view/link/5f5075c763768959e2d109df#map)

### Spring IOC

### 概念

- `IOC`：是一种 `控制反转` 的理念、依赖 `DI` `依赖注入` 来实现
- 用来解决代码耦合

### 举例

- A.java 中, new B()  ->  A.java 中, @Autowired B b

### 过程

#### 标记Bean方式

1. 配置类 xml
2. @注解
3. JavaConfig

#### 加载Spring上下文

- `XML`：new ClassPathXmlApplicationContext("xx.xml")
- `@`：new AnnotationConfigApplicationContext(config.class)

#### getBean

- 通过 context.getBean 获取注入 IOC 容器中的 Bean
- context.getBean() 是 `装饰器模式`，实际上是使用 BeanFactory.getBean()
- BeanFactory.getBean() 是 `简单工厂模式`，通过传入参数获取想要的实例

## ApplicationContext

### 简介

- Bean 生命周期管理容器

### 类构造图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608542463673.png)

## BeanFactory

### 简介

- Spring 的顶层核心接口、负责生产Bean

### 类构造图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608542140240.png)

### 简单工厂模式的实现

- 根据传入唯一标识来获得Bean对象
- 但是在传参前创建、还是在传参后创建该对象，需要按具体情况决定

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608538892530.png)

## BeanDefinition

### 简介

- Spring 的顶层核心接口，封装了生产 Bean 的一切条件要素

- Bean 的统一容器、Bean的定义
- 描述了 Bean 的必要属性 与 各种特制属性
- 所有的Bean，无论是通过 xml、注解、javaConfig，最终都会转载成 BeanDefinition
- 所有的 BeanDefinition，最终都会放入容器 BeanDefinitionMap 中

### 类构造图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608542200127.png)

### 向上联系图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608795496632.png)

- `BeanMetadateElement`：BeanDefinition 元数据，返回该 Bean 的来源
- `AttributeAccessor`：提供对 BeanDefinition 属性操作能力

### 注册顺序

1. @ComponentScan：
   1.  @Component、@Service、@Controller
2. @Import：
   1. @Component、@Service、@Controller
3. @Configuration
   1. 该类里的 @Bean
   2. 该类里 @Import 导入的实现了 @ImportBeanDefinitionRegistrar 接口的
4. @Import：
   1. 导入的 @Configuration
   2. @Configuration 中的 @Bean
   3. @Configuration 中 @Import 进来的实现了 ImportBeanDefinitionRegistrar 接口的
5. @Import：
   1. 导入的实现了 DeffredImportSelector 接口的 @Configuration
   2. 该 @Configuration 中的 @Bean
   3. @Configuration 中 @Import 进来的实现了 ImportBeanDefinitionRegistrar 接口的

## BeanDefinitionRegistry

### 简介

- `作用`：负责将所有的 Beandefinition 注册到 BeanDefefinitionMap 中

### 类构造图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608542261162.png)

## BeanDefinitionReader

### 简介

- 负责读取所有路径下的类

### 类构造图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608542605906.png)

## BeanDefinitionScanner

### 简介

- 负责扫描 BeanDefinitionReader 读取到的类中，被声明的Bean
  - xml
  - 注解
  - javaConfig

## ApplicationContext 和 BeanFactory 的区别

### 共同

- 都具有 getBean，生产 bean 的能力

### 不同

- ApplicationContext 实现了很多的其他服务接口
  - 监听器、国际化、消息、Bean的生命周期管理...
- Bean 只是一个无情的生产Bean机器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608545235118.png)

## Bean的生命周期

### 实例化

- 通过 Beandefinition 中保存的类对象引用，反射实例化
  - `反射`：是由 Spring 控制的
  - `工厂`：是可以自己控制的

### 填充属性

- @Autowired、@Value...

### 初始化

- initMethod()、destroy()...

### 存入容器

- 最终放入一个 Map 容器中
  - 即: `单例池、一级缓存`
  - Key: bean 的 name
  - Value: bean 的 实例
- getBean() 实际上就是去该 Map 中，通过 key 取到对应的 bean 实例

### 关于后置处理器

- `Bean` 的生命周期中，总共会调用9次 BeanPostProcessor

### 关于Bean生命周期的回调函数

- `Bean` 的生命周期 `初始化` 中，会实现很多后缀为 Aware 的接口、作为生命周期过程回调函数

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608605183488.png)

## 循环依赖

### 问题简介

- A @Autowired B
- B @Autowired A
- 以上条件，就会造成在 填充属性 时形成一个闭环

### 解决

- 通过 `三级缓存` 解决、即 3个Map

## BeanFactoryPostProcessor

### 简介

- Bean工厂 的后置处理器
- 可以用来对 Bean工厂 做扩展功能

### 代码示例

```java
public class Demo implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 获取 Bean 定义
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("beanName");

        // 修改 Bean 定义
        beanDefinition.setBeanClassName("updateBeanName");
    }
}
```

## BeanDefinitionRegistryPostProcessor

### 简介

- `作用`：用来新增注册 BeanDefinition

### 代码示例

```java
public class Demo implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        // 获取 Bean 定义
        BeanDefinition beanDefinition = beanFactory.getBeanDefinition("beanName");

        // 修改 Bean 定义
        beanDefinition.setBeanClassName("updateBeanName");
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 新注册一个 BeanDefinition
        registry.registerBeanDefinition("xxx", new AnnotatedGenericBeanDefinition(Demo.class));
    }

}
```

## BeanPostProcessor

### 简介

- `Bean` 的后置处理器
- 是其他框架集成至 Spring 的重要扩展点
- 例: AOP 是在 Bean 生命周期中，初始化完之后，调用后置处理器实现的

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608604757817.png)

## 面试问题

- 描述BeanFactory
  - Spring 的顶层核心接口，专门负责生产Bean
  - 使用简单工厂模式，传入参数返回相应的Bean实例
- BeanFactory 和 ApplicationContext 的区别
  - 相同点:
    - 都是Bean容器，都能生产Bean
  - 不同点:
    - ApplicationContext 不仅实现 BeanFactory，还实现很多其他功能接口
    - 如: 国际化、消息、监听器、Bean生命周期...
- 简述Spring IOC的加载过程
  - `定位`
    - 首先资源文件定位，通常是 XML 或 @注解
    - 一般是在 ApplicationContext 的实现类完成的
      - ClassPathXmlApplicationContext
      - AnnotatitonConfigApplicationContext
    - 将外部资源 读取 为 Resource 类
  - `解析`
    - 对资源文件的解析，在 BeanDefinitionReader 中完成
    - 再通过 BeanDefinitionScanner 扫描出声明的 Bean
    - 比如: 将 XML 委托给 XmlBeanDefiniitionReader 完成
    - 解析结果最终都被封装为 BeanDefinitionHolder
  - `注册`
    - 通过 BeanDefinitionRegistry 将 BeanDefinition 注册到 BeanFactory 容器中的 Map 中
    - Key: Bean 名称、Value: Bean实例、即 BeanDefinitionMap
    - BeanFactory 最常见的实现类为 DefaultListableBeanFactory
  - `扩展`
    - 在 BeanFactory 容器工厂中，可以通过接口实现对 BeanFactory 或 BeanDefinition 进行扩展
    - `BeanFactoryPostProcessor` : Bean工厂后置处理器
    - `BeanDefinitionRegistryPostProcessor` : Bean定义注册后置处理器，常被框架用来集成 Spring
  - `使用`
    - 通过 BeanFactory.getBean() 即可取出 Bean容器 中的 Bean 对象进行使用
    - getBean() 具体就是 Bean 的生命周期了
- 简述Bean的生命周期
  - [参考资料](https://chaycao.github.io/2020/02/15/%E5%A6%82%E4%BD%95%E8%AE%B0%E5%BF%86Spring-Bean%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F/)
  - `实例化`
    - 实例化 Bean 实例
  - `属性赋值`
    - 设置对象属性
  - `初始化`
    - 检查 Aware 相关接口并设置依赖
    - BeanPostProcessor 前置处理函数调用
    - 是否实现 InitializingBean 接口
    - 是否配置自定义的 init-method
    - BeanPostProcessor 后置处理函数调用
  - `销毁`
    - 注册 Destruction 相关回调接口
    - 使用中...
    - 是否实现 DisposableBean 接口
    - 是否配置自定义的 destory-method
- Spring有哪些扩展接口及调用时机
  - 后缀为 PostProcessor 的后置处理器，例如:
    - BeanFactoryPostProcessor 作用于 IOC 注册完容器后
    - BeanDefinitionRegistryPostProcessor 作用于 IOC 注册完容器后
    - BeanPostProccesor 作用与 Bean 的生命周期中
  - 后缀为 Aware 的扩展器，例如:
    - BeanNameAware
    - BeanClassLoaderAware
    - ...
    - 均作用于 Bean 生命周期中 初始化
- 介绍 BeanFactoryPostProcessor 在 Spring 中的用途
  - Bean工厂后置处理器，主要用于读取配置类、便于其他框架接入扩展

## BeanFactory 和 FactoryBean 的区别

[参考资料](https://juejin.cn/post/6844903967600836621)

- BeanFactory 是 Spring 顶层核心接口，只负责生产Bean、提供Bean容器
- FactoryBean 本质上还是一个 Bean，也被 BeanFactory 管理
- 只是对于实现了 FactoryBean 接口的Bean, 在 getBean() 时可以获取更复杂的Bean
- 返回的是 FactoryBean 接口的 getObject() 的对象实例

### 

### 

### 

### 

### 