# 徐庶 - Spring IOC加载流程

## IOC意义图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608780732643.png)

## 第一步: 构造函数的介绍

### 代码示例

```java
public static void main(String[] args)   {
   // 加载spring上下文
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MainConfig.class);

   Car car =  context.getBean("car",Car.class);
   System.out.println(car.getName());
}
```

### AnnotationConfigApplicationContext 的类结构图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608791317814.png)

### 实际调用的IOC容器的有参构造

```java
		/**
     * 可以传入多个 annotatedClasses，但这种情况出现的较少
     *
     * @param annotatedClasses 可变参 注解配置Bean Class
     */
    public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
        /**
         * 调用本类无参构造之前，会先调用父类无参构造，即 GenericApplicationContext 的无参构造函数
         * 父类构造函数工作:
         *      初始化 DefaultListableBeanFactory, 并赋值给 IOC容器属性 beanFactory
         *
         * 本类构造函数工作:
         *      初始化读取器 AnnotatedBeanDefinitionReader
         *      初始化扫描器 ClassPathBeanDefinitionScanner
         *          scanner的作用不大，仅仅用于外部手动调用 .scan 等方法才有用
         */
        this();

        /**
         * 把传入的类进行注册, 存在两种情况:
         *      传入传统配置类
         *      传入Bean(一般没人这么做)
         *
         * 在后面源码中可知，Spring对配置类做了区分:
         *      带 @Configuration 的配置类, 被称之为 FULL 配置类
         *      不带 @Configuration 的配置类, 被称之为 Lite 配置类
         *          例如: @Component, @Import, @ImportResource, @Service, @ComponentScan等注解的配置类
         *          有些地方也把 Lite 配置类称为 普通Bean
         */
        register(annotatedClasses);

        /**
         * 刷新 IOC 容器
         */
        refresh();
    }
```

### 有参构造调用本类的无参构造

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
        
        private final AnnotatedBeanDefinitionReader reader;
        
        private final ClassPathBeanDefinitionScanner scanner;
        
        public AnnotationConfigApplicationContext() {
            // 会隐式调用父类的构造方法，初始化DefaultListableBeanFactory

            // 初始化一个注解bean定义读取器，主要作用是用来读取被注解的了bean
            this.reader = new AnnotatedBeanDefinitionReader(this);

            // 初始化一个扫描器，它仅仅是在我们外部手动调用.scan 等方法才有用
            this.scanner = new ClassPathBeanDefinitionScanner(this);
        }
    }
```

### 无参构造调用父类的无参构造

```java
 public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {

        private final DefaultListableBeanFactory beanFactory;
        
        private ResourceLoader resourceLoader;

        private boolean customClassLoader = false;

        private final AtomicBoolean refreshed = new AtomicBoolean();

        public GenericApplicationContext() {
            // 初始化 Bean工程，用于生产和获得Bean
            this.beanFactory = new DefaultListableBeanFactory();
        }
    }
```

## 第二步: Reader实例化的介绍

- 在 AnnotationConfigApplicationContext 无参构造中，实例化了 AnnotatedBeanDefinitionReader
- 该 Bean 定义读取器主要做两件事:
  - 注册内置 BeanPostProcessor
  - 注册相关 BeanDefinition

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608793959183.png)

### IOC容器调用Reader的有参构造

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
  // 构造入参 registry 即 AnnotationConfigApplicationContext 的实例
  this(registry, getOrCreateEnvironment(registry));
}
```

### 调用同类其他有参构造

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        Assert.notNull(environment, "Environment must not be null");
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
  			
  			// 重点是这一行
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
```

### registerAnnotationConfigProcessors() 的核心代码

```java
// 首先判断容器中是否已经存在 ConfigurationClassPostProcessor Bean
        if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {

            /**
             * 不存在时（这里肯定不存在，因为是IOC容器的首次初始化）
             *      通过 RootBeanDefinition 的有参构造，创建 ConfigurationClassPostProcessor 的 BeanDefinition
             *
             *      RootBeanDefinition 是 BeanDefinition 的子类
             */
            RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);

            def.setSource(source);

            /**
             * 执行 registerPostProcessor() 函数, 进行注册 Bean
             *      当然, 这里注册其他 Bean 也是同样流程
             */
            beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
        }
```

### 实际调用注册后置处理器的代码

```java
private static BeanDefinitionHolder registerPostProcessor(BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {
        /**
         * 这里为 BeanDefinition 设置了一个 Role
         *      ROLE_INFRASTRUCTURE : 代表这是 Spring 内部的，而非用户自定义的
         */
        definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

        /**
         * 这里 registry 的实现类是 DefaultListableBeanFactory
         */
        registry.registerBeanDefinition(beanName, definition);

        return new BeanDefinitionHolder(definition, beanName);
    }
```

### 最终实际注册BeanDefinition的核心代码

```java
 				/**
         * DefaultListableBeanFactory.registerBeanDefinition() 中,
         *      最终将 BeanDefinition 注册进 BeanFactory
         *
         * 实际上: 是往 beanDefinitionMap 中 put 一条数据
         *      key: beanName
         *      value: beanDefinition
         */
        this.beanDefinitionMap.put(beanName, beanDefinition);

        /**
         * 将 beanName 添加到 beanFactory 中存储 BeanName 的集合
         * 
         * 实际上: 是往 bean 工厂里一个 List<String> 的集合属性，添加当前注册的 beanName
         */
        this.beanDefinitionNames.add(beanName);
```

- 到这里就可以看出，DefaultListableBeanFactory 就是我们实际所说的 容器 了
- 这两个属性相当重要，一定要记住
  - beanDefinitionMap
  - beanDefinitionNames

### ConfigurationClassPostProcessor 的介绍

- 上面介绍过，这里会一连串注册好几个 Bean
  - `ConfigurationClassPostProcessor`：这里面**最重要的 Bean**（没有之一）
  - 也是 Spring 体系中极为重要的一个 Bean
- 实现了 BeanDefinitionRegistryPostProcessor 接口
  - 等于实现了 BeanFactoryPostProcessor 接口（**Spring 的重要扩展点之一**）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608796490940.png)

- 除了注册了 ConfigurationClassPostProcessor 之外，还注册了其他 Bean
  - 比如实现了 BeanPostProcessor（**Spring 的重要扩展点之一**）的 Bean
  - ......

## 第三步: Scanner实例化的介绍

- 由于这里的 scanner 仅仅方便程序员手动调用 IOC 容器的 scan 方法
- 重要性比较低，就不再赘述

## 第四步: 注册配置类为 BeanDefinition

```java
// 回到最开始，第二行代码
register(annotatedClasses);
```

### register 最终的核心代码

```java
/**
 * register() 接收的实际是数组
 * 最终会循环调用如下方法
 */
 <T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
                            @Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {
        /**
         * AnnotatedGenericBeanDefinition: 用于描述Bean的结构对象
         *
         * 此处作用:
         *      将 传入的标记了注解的类，转为该结构
         *      该结构中 getMetadata() 函数，可以拿到 类上的注解
         */
        AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);

        /**
         * 判断是否需要跳过注解
         *
         * Spring 中有一个 @Condition 注解
         *      当不满足条件时，这个 bean 就不会被解析
         */
        if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
            return;
        }

        abd.setInstanceSupplier(instanceSupplier);

        /**
         * 解析 Bean 的作用域
         *
         * 如果 Bean 没有设置作用域的话，默认为 单例
         */
        ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
        abd.setScope(scopeMetadata.getScopeName());

        // 获得 bean 的名称
        String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

        /**
         * 解析通用注解, 填充至 AnnotatedGenericBeanDefinition 中
         *
         * 解析注解为:
         *     @Lazy、@Primary、@DependsOn、@Role、@Description
         */
        AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);

        /**
         * 限定符的处理, 并不特指 @Qualifier, 理论上是任何注解, 这里并不判断 注解的有效性
         *
         * 常规方式初始化 Spring IOC, qualifiers 永远都为空, 包括上面的 name 和 instanceSupplier 也为空
         *      即: new AnnotationConfigApplicationContext(xxx.class) 这种方式
         *
         * 但 Spring 提供了其他方式注册 Bean, 就有可能会传入这些属性
         */
        if (qualifiers != null) {
            // 可能传入的是数组, 所以会循环处理
            for (Class<? extends Annotation> qualifier : qualifiers) {
                // 优先处理 @Primary
                if (Primary.class == qualifier) {
                    abd.setPrimary(true);
                }
                // 处理 @Lazy
                else if (Lazy.class == qualifier) {
                    abd.setLazyInit(true);
                }
                // 其他，AnnotatedGenericBeanDefinition有个Map<String,AutowireCandidateQualifier>属性，直接 put
                else {
                    abd.addQualifier(new AutowireCandidateQualifier(qualifier));
                }
            }
        }

        for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
            customizer.customize(abd);
        }

        // 这个方法用处不大，就是把AnnotatedGenericBeanDefinition数据结构和beanName封装到一个对象中
        BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);

        definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);

        /**
         * 最终调用 DefaultListableBeanFactory.registerBeanDefinition() 进行注册
         * 即维护: beanDefinitionMap & beanDefinitionNames
         */
        BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
    }
```

- 以 常规方式 注册的配置类，在此方法中，除了第一个参数，其他参数都是默认值

### registerBeanDefinition 代码解析

```java
		/**
     * 这个方法在上面 Spring 注册 内置Bean 的时候已经解析过了
     */
    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // 获取beanName
        String beanName = definitionHolder.getBeanName();

        // 注册bean
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // Spring支持别名
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

- 到这里，注册配置类也分析完毕了

## 第五步: IOC容器的refresh()

- 到目前为止，Spring 还没有进行扫描，还只做了如下工作:
  - 实例化一个 bean 工厂
  - 注册一些内置 baen
  - 注册入参的配置类 bean
- 真正的大头都在第三行代码
  - **refresh()**

### refresh() 的解读

```java
public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {

            // 刷新预处理，和主流程关系不大，就是保存了容器的启动时间，启动标志等
            prepareRefresh();

            /**
             * 和主流程关系也不大，最终获得了 DefaultListableBeanFactory
             *      DefaultListableBeanFactory 实现了 ConfigurableListableBeanFactory
             *
             * XML模式下，会在这里读取 BeanDefinition
             */
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            /**
             * 这里仍是做一些 准备工作, 添加了两个 后置处理器
             *      ApplicationContextAwareProcessor
             *      ApplicationListenerDetector
             *
             * 还设置了 忽略自动装配 和 允许自动装配 的接口
             *      如果不存在某个 Bean 的时候，Spring 就会自动注册 Singleton Bean
             *
             * 还设置了 Bean 表达式解析器 ...
             */
            prepareBeanFactory(beanFactory);

            try {
                // 这是一个空方法
                postProcessBeanFactory(beanFactory);

                // 执行自定义的 BeanFactoryPostProcessor 和内置的 BeanFactoryPostProcessor
                invokeBeanFactoryPostProcessors(beanFactory);

                // 注册BeanPostProcessor
                registerBeanPostProcessors(beanFactory);

                // 为 IOC 容器初始化 消息源
                initMessageSource();

                // 为 IOC 容器初始化 事件广播器
                initApplicationEventMulticaster();

                // 空方法
                onRefresh();

                // 为 IOC 容器注册 监听器数组
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```

### prepareBeanFactory() 的作用

```java
 protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 设置类加载器
        beanFactory.setBeanClassLoader(getClassLoader());

        // 设置bean表达式解析器
        beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));

        // 属性编辑器支持
        beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

        // 添加一个后置处理器：ApplicationContextAwareProcessor，此后置处理处理器实现了BeanPostProcessor接口
        beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

        // 以下接口，忽略自动装配
        beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
        beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
        beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
        beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
        beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

        // 以下接口，允许自动装配,第一个参数是自动装配的类型，，第二个字段是自动装配的值
        beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
        beanFactory.registerResolvableDependency(ResourceLoader.class, this);
        beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
        beanFactory.registerResolvableDependency(ApplicationContext.class, this);

        // 添加一个后置处理器：ApplicationListenerDetector，此后置处理器实现了BeanPostProcessor接口
        beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

        // Detect a LoadTimeWeaver and prepare for weaving, if found.
        if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            // Set a temporary ClassLoader for type matching.
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }

        // 如果没有注册过bean名称为XXX的bean，Spring就自己创建一个名称为XXX的singleton bean
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

### 重点: invokeBeanFactoryPostProcessors()

[invokeBeanFactoryPostProcessors流程图](https://www.processon.com/view/link/5f18298a7d9c0835d38a57c0)

```java
		/**
     * 重点方法!
     */
    protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        
        /**
         * getBeanFactoryPostProcessors() 不是永远的空集合（前人采坑，后人乘凉）
         * Spring 允许开发人员 手动添加 BeanFactoryPostProcessor
         *      即: AnnotationConfigApplicationContext.addBeanFactoryPostProcessor(xxx);
         */
        PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

        if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
        }
        
    }
```

#### 坑: getBeanFactoryPostProcessors()

```java
/**
 * 因为这个 beanFactoryPostProcessors 属性，在 Spring 中没有代码添加元素
 * 而是留白给开发人员使用
 */
public List<BeanFactoryPostProcessor> getBeanFactoryPostProcessors() {
		return this.beanFactoryPostProcessors;
	}
```

#### 实际调用的 invokeBeanFactoryPostProcessors()

```java

```

