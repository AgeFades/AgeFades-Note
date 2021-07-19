[TOC]

# 徐庶 - SpringBoot自动装配

## 资料

[SpringBoot自动装配流程图](https://www.processon.com/view/link/5fc0abf67d9c082f447ce49b)

[参考链接](https://juejin.cn/post/6927268163487072269)

## @SpringBootApplication

- 标记该类为主配置类
- 运行该类 main() 启动整个 SpringBoot 应用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1617953147950.png)

### 注解说明

#### @Target

- 描述注解的使用范围
  - `CONSTRUCTOR`：描述构造器
  - `FIELD`：描述字段
  - `LOCAL_VARIABLE`：描述局部变量
  - `METHOD`：描述方法
  - `PACKAGE`：描述包
  - `PARAMETER`：描述参数
  - `TYPE`：描述类、接口、枚举

#### @Retention

- 标记注解会被保留到哪个阶段
  - `RUNTIME`：被 JVM 保留，在运行时能被 JVM 或 其他反射机制读取和使用
  - `CLASS`：编译时保留，class文件中存在，JVM中忽略
  - `SOURCE`：源码保留，编译忽略

#### @Documented

- 生成 Java Doc

#### @Inherited

- 标记会被自动继承（子类继承父类所有属性）

#### @SpringBootConfiguration

- 标记为 SpringBoot 配置类

#### @EnableAutoconfiguration

- 标记开启自动装配

#### @ComponentScan

- 扫描包（扫描指定路径Bean，加载至IOC）
- 未指定时，自动扫描 `当前配置类所有同级及子级` 的包路径
  - `TypeExcludeFilter`：自定义排除 SpringBoot 提供的扩展类
  - `AutoConfigurationExcludeFilter`：过滤会自动装配的配置类，避免重复

## @EnableAutoConfiguration

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626318668294.png)

### @AutoConfigurationPackage

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626318776193.png)

- 将当前配置类所在包保存在 BasePackages 的Bean中，供Spring内部使用
- 就是注册了一个保存当前配置类所在包的一个Bean

### @Import(AutoConfigurationImportSelector.class)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626319391050.png)

- 导入 AutoConfigurationImportSelector 实现了 **DeferredImportSelector** 
- Spring内部在解析时，会调用其 **getAutoConfigurationEntry()**

#### 扫描带有META-INF/spring.factories文件的Jar包

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
      return EMPTY_ENTRY;
    }
  
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
  
  	// 从 spring.factories 中获得候选的自动装配类
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
  
  	// 排重
    configurations = removeDuplicates(configurations);
  
    // 根据 EnableAutoConfiguration 注解中属性，获取不需要自动装配的类名单
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
  
    // 进行排除的类：
  	// 1. @EnableAutoConfiguration.exclude
  	// 2. @EnableAutoConfiguration.excludeName
  	// 3. spring.autoconfigure.exclude
    checkExcludedClasses(configurations, exclusions);
  
  	// exclusions 也排除
    configurations.removeAll(exclusions);
  	
  	// 通过读取 spring.factories 进行过滤
  	// 1. OnBeanCondition
  	// 2. OnClassCondition
  	// 3. OnWebApplicationCondition
    configurations = getConfigurationClassFilter().filter(configurations);
  
  	// 调用实现 AutoConfigurationImportListener 的 bean...
  	// 分别把候选的配置名单、排除的配置名单，传进去做扩展
    fireAutoConfigurationImportEvents(configurations, exclusions);
  
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626319926133.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626319959474.png)

- 每个SpringBoot应用，都会引入 spring-boot-autoconfigure

  - 而 spring.factories 文件就在该包下面

- spring.factories 文件是 KV 形式，多个 value 时用 , 隔开

  - 文件中定义了关于 初始化、监听器 等信息

- 而真正使自动配置生效的 key 是 org.springframework.boot.autoconfigure.EnableAutoConfiguration

  - 等同于 @Import({

  - ```java
    # Auto Configure
    org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
    ...省略
    org.springframework.boot.autoconfigure.websocket.WebSocketMessagingAutoConfiguration,\
    org.springframework.boot.autoconfigure.webservices.WebServicesAutoConfiguration
    ```

  - })

- 每个 xxxAutoConfiguration 类都是容器中的一个组件，都加入到容器中，用它们来做自动配置

  - [所有自动配置类表](https://docs.spring.io/spring-boot/docs/current/reference/html/auto-configuration-classes.html#auto-configuration-classes)

- @EnabelAutoConfiguration 注解通过 @SpringBootApplication 被简介标记在 SpringBoot 的启动类上

  - 在 SpringApplication.run(...) 方法内部，就会执行 selectImports() 方法
  - 找到所有 JavaConfig 自动配置类的全限定名对应的 Class 并加载到 Spring IOC  容器中

## 举例解释自动装配原理

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626331998548.png)

- HttpEncodingAutoConfiguration：Http编码自动配置

### @Configuration(proxyBeanMethods = false)

- Spring 会给标记 @Configuration 的配置类创建 cglib动态代理
  - 防止每次调用本类的@bean方法重复创建对象，Bean默认单例

### @EnableConfigurationProperties(ServerProperties.class)

- 启用可以在配置类设置属性对应的类

### @Conditional

- 当 @Conditional 指定的条件成立，才给容器中添加组件，配置里面的内容才生效

| @Conditional扩展注解作用        | （判断是否满足当前指定条件）                     |
| ------------------------------- | ------------------------------------------------ |
| @ConditionalOnJava              | 系统的java版本是否符合要求                       |
| @ConditionalOnBean              | 容器中存在指定Bean；                             |
| @ConditionalOnMissingBean       | 容器中不存在指定Bean；                           |
| @ConditionalOnExpression        | 满足SpEL表达式指定                               |
| @ConditionalOnClass             | 系统中有指定的类                                 |
| @ConditionalOnMissingClass      | 系统中没有指定的类                               |
| @ConditionalOnSingleCandidate   | 容器中只有一个指定的Bean，或者这个Bean是首选Bean |
| @ConditionalOnProperty          | 系统中指定的属性是否有指定的值                   |
| @ConditionalOnResource          | 类路径下是否存在指定资源文件                     |
| @ConditionalOnWebApplication    | 当前是web环境                                    |
| @ConditionalOnNotWebApplication | 当前不是web环境                                  |
| @ConditionalOnJndi              | JNDI存在指定项                                   |

### debug查看配置类的生效与否

- 配置文件中启用 debug = true

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626332739574.png)

### 分析HttpEncodingAutoConfiguration

1. @Configuration：标注为配置类，会被加载到IOC容器
2. @ConditionalOnWebApplication：标注该配置类只有在Web环境下生效
3. @ConditionOnClass：标注该配置类只有在项目中有CharacterEncodingFilter 才生效
   - CharacterEncodingFilter：SpringMVC中解决乱码的过滤器
4. @ConditionalOnProperty：标注该配置类只有在 spring.http.encoding.enabled = true 时才生效
   - spring.http.encoding.enbaled 不存在配置也是生效的
5. @EnbaleConfigurationProperties：将配置文件中值绑定的属性类加载到 IOC 容器

![](https://note.youdao.com/yws/public/resource/8805e97f26dc9661bc2ecdbf5ca22393/xmlnote/94237F114C6445B2B49E713EAFA38C2B/14482)

## 源码分析

### 1. 启用自动配置

- @SpringBootApplication 注解上带有该配置
  - 表示启用自动配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626588998213.png)

#### 1. 导入自动装配选择器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589094668.png)

#### 2. 实现延时导入选择器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589193177.png)

#### 3. 实现导入Bean分组方法

- 对项目业务定义的bean和三方框架的bean做分组，方便后面排序
  - 先处理项目中的Bean
  - 再处理三方框架导入的自动配置Bean
- 这么做的意义，举例：
  - mybatis 的 starter 导入了 SqlSessionFactory Bean
    - Bean 上标注了 @ConditionOnBean 的条件
  - 项目自定义了一个 SqlSessionFactory Bean
  - 根据分组的加载顺序，就会优先加载项目自定义的 SqlSessionFactory
  - 而在加载 starter 中的 SqlSessionFactory 时，就会根据 @ConditionOnBean 的条件决定不加载，采用项目自定义的 SqlSessionFactory

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589273844.png)

#### 4. 调用Group实现类的process()

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589415930.png)

##### 1. 获取所有自动配置类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589880468.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626589916322.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626590008550.png)

##### 2. spring.factories举例

- KV形式的键值对

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626590047294.png)

##### 3. 加载是启用自动配置Key对应的配置类

- 即加载上图示例中 org.springframework.boot.autoconfigure.EnableAutoConfiguration 对应的Bean集合
- spring-boot-autoconfigure 官方下的自动配置类就有127个(不同版本数量不一样)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626590174546.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626590188389.png)

##### 4. 对所有自动配置类排重及过滤

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592182009.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592276412.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592291147.png)

##### 5. 配置文件中的排除Bean方式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592611027.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592708156.png)

##### 6. 返回最终项目有效自动配置类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626592777225.png)

#### 5. 调用Group实现类的selectImports()

- 对上步返回的项目最终有效自动配置类进行排序（只会对当前组里的Bean排序）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626662708024.png)



## 自定义Starter

### 参照案例

- WebMvcAutoConfiguration

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626575871248.png)

### 操作案例

- 创建 pom 项目 test-spring-boot-starter
  - 创建子model spring-boot-starter-test
    - 依赖 spring-boot-starter-test-autoconfiguration
    - 空jar
  - 创建子model spring-boot-starter-test-autoconfiguration
    -  写个 IndexController Bean，放在 spring.factories
  - mvn clean install 下载到本地Maven库
- 业务项目依赖 spring-boot-starter-test
  - 启动项目，访问 IndexController
- 这就是自定义starter、自动配置的基本流程 

## @Component 之类和 @Import 的区别

- @Component、@Service、@Controller.. 标注的类一定是能项目能扫描到的路径
  - @ComponentScan 即项目扫描bean的路径
- @Import 可以导入项目扫描不到的类、还可以批量导入Bean

## Aop自动配置简讲

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1626663112403.png)

