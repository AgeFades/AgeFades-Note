[TOC]

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
public static void invokeBeanFactoryPostProcessors(
            ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

        // 初始化准备收集 处理后的Bean Set集合
        Set<String> processedBeans = new HashSet<>();

        /**
         * 因为 beanFactory 实际是 DefaultListableBeanFactory, 即 BeanDefinitionRegistry 的实现类
         * 所以此处肯定满足 if
         */
        if (beanFactory instanceof BeanDefinitionRegistry) {
            BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;

            // 初始化准备收集 BeanFactoryPostProcessor List集合
            List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();

            /**
             * 初始化准备收集 BeanDefinitionRegistryPostProcessor List集合
             *      BeanDefinitionRegistryPostProcessor 扩展了 BeanFactoryPostProcessor
             */
            List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

            /**
             * 循环处理入参 beanFactoryPostProcessors
             * 正常情况下, beanFactoryPostProcessors 肯定没有数据
             *      因为这是 开发人员手动添加的，而非 Spring 扫描的
             */
            for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
                /**
                 * 如果是实现了 BeanDefinitionRegistryPostProcessor 的扩展 bean工厂后置处理器
                 *      直接执行 postProcessBeanDefinitionRegistry
                 *      然后将对象收集至 registryProcessors 容器集合
                 */
                if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                    BeanDefinitionRegistryPostProcessor registryProcessor =
                            (BeanDefinitionRegistryPostProcessor) postProcessor;
                    registryProcessor.postProcessBeanDefinitionRegistry(registry);
                    registryProcessors.add(registryProcessor);
                }

                else {
                    /**
                     * 非实现 BeanDefinitionRegistryPostProcessor 的 bean工厂后置处理器，
                     *      直接将对象收集至 regularPostProcessors 容器集合
                     */
                    regularPostProcessors.add(postProcessor);
                }
            }

            /**
             * 一个临时变量，用来装载BeanDefinitionRegistryPostProcessor
             */
            List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

            /**
             * 收集 beanName 数组, bean 类型为 BeanDefinitionRegistryPostProcessor
             *      坑: 自己创建一个实现 BeanDefinitionRegistryPostProcessor 接口的类，并标注为 @Component
             *          配置类也标注为 @Component，但这里却还是没有拿到
             *
             *      原因: 因为直到这一步, Spring 也没有去扫描
             *          扫描是在 ConfigurationClassPostProcessor(非常重要的类) 类中完成
             *          即下面的第一个 invokeBeanDefinitionRegistryPostProcessors() 函数
             *
             */
            String[] postProcessorNames =
                    beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

            for (String ppName : postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    /**
                     * 获得 ConfigurationClassPostProcessors 类, 并收集至 currentRegistryProcessors 集合中
                     *
                     * 该类用于:
                     *      扫描Bean、@Import、@ImportResource ...等各种操作
                     *      用来处理配置类（传统配置类、普通Bean）的各种逻辑
                     */
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));

                    // 把name放到processedBeans，后续会根据这个集合来判断处理器是否已经被执行过了
                    processedBeans.add(ppName);
                }
            }

            // 处理排序
            sortPostProcessors(currentRegistryProcessors, beanFactory);

            /**
             * BeanDefinitionRegistryPostProcessor 两个集合的合并
             * 为什么要合并？
             *
             * 因为: 开始时，Spring只会执行 这些元素的独有方法，而不会执行 这些元素的父类方法
             *      即: BeanFactoryPostProcessors 的方法
             *
             * 所以: 将这些后置处理器合并至一个集合中，便于 后续统一执行父类方法
             */
            registryProcessors.addAll(currentRegistryProcessors);

            /**
             * 可以理解为执行 ConfigurationClassPostProcessor 的 postProcessBeanDefinitionRegistry 方法
             * Spring 热插拔的一种体现.
             *
             * ConfigurationClassPostProcessor 就相当于一个组件，Spring 很多事情都是交由组件管理
             * 如果不想用这个组件，直接把 注册组件的那一步 去掉即可
             */
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);

            // 因为 currentRegistryProcessors 是一个临时变量，所以需要清除
            currentRegistryProcessors.clear();
  
            // 再次收集 beanName 数组, bean 类型仍为 BeanDefinitionRegistryPostProcessor
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                
                // 遍历判断该 beanName 是否已经被执行过、是否实现 Ordered 接口
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    
                    /**
                     * 既没有被执行过, 也没有实现 Ordered 接口:
                     *      当前对象添加至 currentRegistryProcessors 集合中
                     *      beanName 添加至 processedBeans
                     *      
                     * 反之:
                     *     此处不对该对象做处理，后续再做处理
                     */
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    
                }
            }

            // 处理排序
            sortPostProcessors(currentRegistryProcessors, beanFactory);

            // 合并Processors
            registryProcessors.addAll(currentRegistryProcessors);

            // 执行我们自定义的 BeanDefinitionRegistryPostProcessor
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);

            // 清空临时变量
            currentRegistryProcessors.clear();
            
            /**
             * 上面代码执行了实现 Ordered 接口的 BeanDefinitionRegistryPostProcessor
             * 
             * 那么下面的代码就执行了没有实现 Ordered 接口的 BeanDefinitionRegistryPostProcessor
             */
            boolean reiterate = true;
            while (reiterate) {
                reiterate = false;
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                for (String ppName : postProcessorNames) {
                    if (!processedBeans.contains(ppName)) {
                        currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                        processedBeans.add(ppName);
                        reiterate = true;
                    }
                }
                sortPostProcessors(currentRegistryProcessors, beanFactory);
                registryProcessors.addAll(currentRegistryProcessors);
                invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
                currentRegistryProcessors.clear();
            }
            
            // 上面的代码是执行子类独有的方法，这里需要再把父类的方法也执行一次
            invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);

            // regularPostProcessors装载BeanFactoryPostProcessor，执行BeanFactoryPostProcessor的方法
            // 但是regularPostProcessors一般情况下，是不会有数据的，只有在外面手动添加BeanFactoryPostProcessor，才会有数据
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
        }

        else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
        }
        
        // 找到BeanFactoryPostProcessor实现类的BeanName数组
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

        // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
        // Ordered, and the rest.
        List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
        List<String> orderedPostProcessorNames = new ArrayList<>();
        List<String> nonOrderedPostProcessorNames = new ArrayList<>();
        
        // 循环BeanName数组
        for (String ppName : postProcessorNames) {
            
            // 如果这个Bean被执行过了，跳过
            if (processedBeans.contains(ppName)) {
                // skip - already processed in first phase above
            }
            
            // 如果实现了PriorityOrdered接口，加入到priorityOrderedPostProcessors
            else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            }
            
            // 如果实现了Ordered接口，加入到orderedPostProcessorNames
            else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            }
            
            // 如果既没有实现PriorityOrdered，也没有实现Ordered。加入到nonOrderedPostProcessorNames
            else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }

        // 排序处理priorityOrderedPostProcessors，即实现了PriorityOrdered接口的BeanFactoryPostProcessor
        sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
        
        // 执行priorityOrderedPostProcessors
        invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

        // 执行实现了Ordered接口的BeanFactoryPostProcessor
        List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : orderedPostProcessorNames) {
            orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        sortPostProcessors(orderedPostProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

        // 执行既没有实现PriorityOrdered接口，也没有实现Ordered接口的BeanFactoryPostProcessor
        List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
        for (String postProcessorName : nonOrderedPostProcessorNames) {
            nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

        // Clear cached merged bean definitions since the post-processors might have
        // modified the original metadata, e.g. replacing placeholders in values...
        beanFactory.clearMetadataCache();
    }
```

1. 首先判断 beanFactory 是否为 BeanDefinitionRegistry 的实例（肯定是），然后执行如下操作

2. 定义一个 Set 集合 `processedBeans`，装载 beanName

   1. 后续会根据该 Set，判断后置处理器是否被执行过

3. 定义两个 List 集合

   1. `regularPostProcessors`：用来装载 BeanFactoryPostProcessors
   2. `registryProcessors`：用来装载 BeanDefinitionRegistryPostProcessor
      1. `BeanDefinitionRegistryPostProcessors` 扩展了 BeanFactoryPostProcessor
      2. `BeanDefinitionRegistryPostProcessors` 的两个重要方法
         1. 独有的 `postPrcoessBeanDefinitionRegistry()`
         2. 父类的 `postProcessBeanFactory()`

4. 遍历传入的 beanFactoryPostProcessors（除非手动添加，否则永远都是空集合）

   1. 判断遍历的 postProcessor 是不是 BeanDefinitionRegistryPostProcessor
      1. `是`
         1. 执行 postProcessor 的 `postPrcoessBeanDefinitionRegistry()`
         2. 添加至 `registryProcessors ` 集合
      2. `否`
         1. 添加至 `regularPostProcessors` 集合

5. 定义一个临时集合变量 `currentRegistryProcessors`

   1. 用于装载 BeanDefinitionRegistryPostProcessor

6. 收集 beanName 数组 `postProcessorNames`

   1. 从 beanDefinitionNames 遍历收集 `BeanDefinitionRegistryPostProcessor.class` 类型的 beaName
   2. 一般情况下，只会找到一个元素，即 `ConfigurationAnnotationProcessor`
      1. 直到这一步，Spring 还没有进行扫描，扫描是在 `ConfigurationAnnotationProcessor` 中完成的
      2. 即: 下面的第一个 `invokeBeanDefinitionRegistryPostProcessors()`

7. 遍历 `postProcessorNames`，其实一般就是 `ConfigurationAnnotationProcessor`

   1. 判断是否实现 PriorityOrdered 接口（`ConfigurationAnnotationProcessor`实现了）
   2. 添加至 `currentRegistryProcessors` 临时变量集合
   3. 添加至 `processedBeans` （现在还没处理，但是马上处理）

   ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608866424025.png)

8. 进行排序

   1. 实现了 PriorityOrdered 接口的，就说明此 后置处理器 是有序的，需要进行排序
   2. 不过现在仍然只有一个  `ConfigurationAnnotationProcessor`

9. 将 `currentRegistryProcessors` 合并至 `registryProcessors`

   1. 合并的理由：
      1. 目前为止，Spring只执行了 BeanDefinitionRegistryPostProceessor 的独有方法
      2. 将这些对象合并至一个容器，是为了后续便于统一执行其父类方法
      3. 即: BeanFactoryPostProcessor 接口中的方法
      4. 不过现在仍然只有一个  `ConfigurationAnnotationProcessor`

10. 现在开始执行 `ConfigurationAnnotationProcessor` 中的 `postPrcoessBeanDefinitionRegistry()`

    1. 这里是 Spring 热插拔的设计思想体现，很多功能是交由 插件 处理
    2. 这里的 后置处理器 就相当于一个插件，想扩展则添加 后置处理器 即可
    3. `ConfigurationAnnotationProcessor` 这里作用就是完成 Spring 的扫描

    ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608866987440.png)

11. 清空 `currentRegistryProcessors`

    1. 该临时变量当前阶段任务已完成，后续仍需使用，所以进行 清空

12. 再次收集 beanName 数组并重新赋值给 `postProcessorNames`

    1. 从 beanDefinitionNames 遍历收集 `BeanDefinitionRegistryPostProcessor.class` 类型的 beaName
    2. 遍历判断该 后置处理器 是否被执行过了
       1. 既没有被执行过，又实现了 Ordered 接口：
          1. 添加至 `currentRegistryProcessors`
          2. 添加至 `postProcessorNames`
       2. 否则：
          1. 这里不做处理，留给下面处理
    3. 到这里就可以获取到开发人员标记 @Component 的后置处理器
    4. `注意`：
       1. `ConfigurationAnnotationProcessor` 在上面已经被执行过了，所以不会再添加至 上面 这两个容器

13. 处理排序

14. 再次将 `currentRegistryProcessors` 合并至 `registryProcessors`，理由与上面一致

15. 执行开发人员自定义的 `BeanDefinitionResitryPostProcessor`

16. 清空临时变量

17. 执行上面没有实现 Ordered 接口的  `BeanDefinitionRegistryPostProcessor`

18. 上面代码已将 子类独有方法 全部执行完毕，这里统一执行 父类方法

19. 执行 `regularPostProcessors` 集合中 后置处理器 方法

    1. 一般不会有数据，只有开发人员手动添加 BeanFactoryPostProcessor 时才有数据

20. 查找实现了 BeanFactoryPostProcessor 的后置处理器，并执行其方法

#### 重点: ConfigurationAnnotationProcessor 的 postPrcoessBeanDefinitionRegistry()

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608867832166.png)

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        List<BeanDefinitionHolder> configCandidates = new ArrayList<>();

        // 收集所有 BeanDefinition 的 beanName 至 candidateNames 数组
        String[] candidateNames = registry.getBeanDefinitionNames();

        // 遍历 candidateNames 数组
        for (String beanName : candidateNames) {

            // 根据 beanName 获得 BeanDefinition
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            
            /**
             * 对于Lite配置类:
             *      getBean() 得到的就是原本的那个配置类
             *      
             * 对于Full配置类:
             *      getBean() 得到的是被 Cglib 代理的类、Cglib底层依赖 ASM 框架
             * 			比如: 
             *				@Configuration 标记 Java 类 RedisConfig
             *				RedisConfig 中两个 @Bean
             *						@Bean A 中属性设置依赖 @Bean B
             * 				此时被 @Configuration 标记，得到 Cglib 动态代理类
             * 				@Bean A 设置属性时，就会先去 beanFactory 中 getBean(B)
             *				这样得到的就是单例 bean，而不是每次重复 new 的对象
             * 			
             * 			所以:
             *				Cglib 代理类调用本类方法时，是会进入代理class路由，调用的是被增强的代理类的方法
             *				JDK 代理类调用本类方法时，不会再走一遍增强代理类
             */
            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                    ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
                }
            }
            
            /**
             * 判断 bean 是 Full配置类 ？ 还是 Lite配置类？并进行标记
             * 满足条件的，添加至 configCandidates
             */
            else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        // 如果没有配置类，直接返回
        if (configCandidates.isEmpty()) {
            return;
        }
        
        // 处理排序
        configCandidates.sort((bd1, bd2) -> {
            int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
            int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
            return Integer.compare(i1, i2);
        });

        // Detect any custom bean name generation strategy supplied through the enclosing application context
        SingletonBeanRegistry sbr = null;
        
        // DefaultListableBeanFactory 最终会实现 SingletonBeanRegistry 接口，所以可以进入到这个if
        if (registry instanceof SingletonBeanRegistry) {
            sbr = (SingletonBeanRegistry) registry;
            if (!this.localBeanNameGeneratorSet) {
                
                // spring中可以修改默认的bean命名方式，这里就是看用户有没有自定义bean命名方式，虽然一般没有人会这么做
                BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
                if (generator != null) {
                    this.componentScanBeanNameGenerator = generator;
                    this.importBeanNameGenerator = generator;
                }
                
            }
        }

        if (this.environment == null) {
            this.environment = new StandardEnvironment();
        }

        // Parse each @Configuration class
        ConfigurationClassParser parser = new ConfigurationClassParser(
                this.metadataReaderFactory, this.problemReporter, this.environment,
                this.resourceLoader, this.componentScanBeanNameGenerator, registry);

        Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
        Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
        do {
            // 解析配置类（传统意义上的配置类或者是普通bean，核心来了）
            parser.parse(candidates);
            parser.validate();

            Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);

            // Read the model and create bean definitions based on its content
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                        registry, this.sourceExtractor, this.resourceLoader, this.environment,
                        this.importBeanNameGenerator, parser.getImportRegistry());
            }
            
            // 直到这一步才把Import的类，@Bean @ImportResource 转换成BeanDefinition
            this.reader.loadBeanDefinitions(configClasses);

            // 把configClasses加入到alreadyParsed，代表已经被解析过
            alreadyParsed.addAll(configClasses);

            candidates.clear();
            
            /**
             * 获得注册器里面 BeanDefinition 的数量 和 candidateNames 进行比较
             * 如果大于的话，说明有新的 BeanDefinition 注册进来了
             */
            if (registry.getBeanDefinitionCount() > candidateNames.length) {

                // 从注册器里面获得 BeanDefinitionNames
                String[] newCandidateNames = registry.getBeanDefinitionNames();

                // candidateNames 转换set
                Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
                Set<String> alreadyParsedClasses = new HashSet<>();
                
                // 循环alreadyParsed。把类名加入到alreadyParsedClasses
                for (ConfigurationClass configurationClass : alreadyParsed) {
                    alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
                }
                for (String candidateName : newCandidateNames) {
                    if (!oldCandidateNames.contains(candidateName)) {
                        BeanDefinition bd = registry.getBeanDefinition(candidateName);
                        if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                                !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                            candidates.add(new BeanDefinitionHolder(bd, candidateName));
                        }
                    }
                }
                candidateNames = newCandidateNames;
            }
        }
        while (!candidates.isEmpty());

        // Register the ImportRegistry as a bean in order to support ImportAware @Configuration classes
        if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
            sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
        }

        if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
            // Clear cache in externally provided MetadataReaderFactory; this is a no-op
            // for a shared cache since it'll be cleared by the ApplicationContext.
            ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
        }
    }
```

1. 获得所有 beanName, 放入 `candidateNames` 数组
2. 遍历 `candidateNames` 数组
   1. 根据 beanName 获得 BeanDefinition
   2. 判断此 BeanDefinition 是否已经被处理过
3. 判断是否为配置类
   1. `是`
      1. 加入 `configCandidates` 数组
   2. 判断时，还会标记 Full配置类 或 Lite配置类
4. 如果没有配置类，直接返回
5. 处理排序
6. **解析配置类**（核心方法）
7. 因为第6步只注册了部分 bean
   1. 这里对剩余 bean 进行统一注册，如 @Import、@Bean...

#### 解析配置类核心代码

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
        this.deferredImportSelectors = new LinkedList<>();
        
        // 循环传进来的配置类
        for (BeanDefinitionHolder holder : configCandidates) {
            // 获得BeanDefinition
            BeanDefinition bd = holder.getBeanDefinition();
            try {
                // 如果获得BeanDefinition是AnnotatedBeanDefinition的实例
                if (bd instanceof AnnotatedBeanDefinition) {
                    parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
                } else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                    parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
                } else {
                    parse(bd.getBeanClassName(), holder.getBeanName());
                }
            } catch (BeanDefinitionStoreException ex) {
                throw ex;
            } catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
            }
        }

        // 执行DeferredImportSelector
        processDeferredImportSelectors();
    }
```

- 因为目前配置类的 BeanDefinition 基本是 AnnotatedBeanDefinition 的实例
- 所以进入第一个 if 中的 parse()

```java
protected final void parse(AnnotationMetadata metadata, String beanName) throws IOException {
        processConfigurationClass(new ConfigurationClass(metadata, beanName));
    }
    
    protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {

        // 判断是否需要跳过
        if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
            return;
        }

        ConfigurationClass existingClass = this.configurationClasses.get(configClass);
        if (existingClass != null) {
            if (configClass.isImported()) {
                if (existingClass.isImported()) {
                    existingClass.mergeImportedBy(configClass);
                }

                return;
            } else {
                this.configurationClasses.remove(configClass);
                this.knownSuperclasses.values().removeIf(configClass::equals);
            }
        }

        SourceClass sourceClass = asSourceClass(configClass);
        do {
           // 重点方法
            sourceClass = doProcessConfigurationClass(configClass, sourceClass);
        }
        while (sourceClass != null);

        this.configurationClasses.put(configClass, configClass);
    }
```

#### 重点: doProcessConfigurationClass()

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608877709710.png)

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
            throws IOException {

        // 递归处理内部类，一般不会写内部类
        processMemberClasses(configClass, sourceClass);

        // 处理 @PropertySource，用来加载 properties 文件
        for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
                sourceClass.getMetadata(), PropertySources.class,
                org.springframework.context.annotation.PropertySource.class)) {
            if (this.environment instanceof ConfigurableEnvironment) {
                processPropertySource(propertySource);
            } else {
                logger.warn("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                        "]. Reason: Environment must implement ConfigurableEnvironment");
            }
        }

        /**
         * 获得 @ComponentScan 具体内容
         *      除了最常用的 basePackage 之外，
         *      还有 includeFilters、excludeFilters ...
         */
        Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
                sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);

        // 如果没有打上 @ComponentScan，或者被 @Condition 条件跳过，就不再进入这个if
        if (!componentScans.isEmpty() &&
                !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {

            // 循环处理componentScans
            for (AnnotationAttributes componentScan : componentScans) {

                /**
                 * componentScan 就是 @ComponentScan 上的具体内容
                 *
                 * sourceClass.getMetadata().getClassName() 就是配置类的名称
                 */
                Set<BeanDefinitionHolder> scannedBeanDefinitions =
                        this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());

                for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                    BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                    if (bdCand == null) {
                        bdCand = holder.getBeanDefinition();
                    }
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                        // 递归调用，因为可能组件类有被 @Bean 标记的方法，或者组件类本身也有 @ComponentScan 等注解
                        parse(bdCand.getBeanClassName(), holder.getBeanName());
                    }
                }
            }
        }

        /**
         * 处理 @Import
         * @Import 是 Spring 中很重要的一个注解，SpringBoot 大量应用该注解
         *
         * @Import 有三种类:
         *      Import 普通类
         *      Import ImportSelector
         *      Import ImportBeanDefinitionRegistrar
         *
         * getImports(sourceClass) 是获得 import 的内容，返回的是一个 set
         */
        processImports(configClass, sourceClass, getImports(sourceClass), true);

        // 处理 @ImportResource
        AnnotationAttributes importResource =
                AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
        if (importResource != null) {
            String[] resources = importResource.getStringArray("locations");
            Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
            for (String resource : resources) {
                String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                configClass.addImportedResource(resolvedResource, readerClass);
            }
        }
        
        /**
         * 处理 @Bean 标注的方法
         * 不会马上转换成 BeanDefinition，而是先用一个 Set 接收
         */
        Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
        for (MethodMetadata methodMetadata : beanMethods) {
            configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
        }
        
        processInterfaces(configClass, sourceClass);
        
        if (sourceClass.getMetadata().hasSuperClass()) {
            String superclass = sourceClass.getMetadata().getSuperClassName();
            if (superclass != null && !superclass.startsWith("java") &&
                    !this.knownSuperclasses.containsKey(superclass)) {
                this.knownSuperclasses.put(superclass, configClass);

                return sourceClass.getSuperClass();
            }
        }

        return null;
    }
```

1. 递归处理内部类（一般不会使用内部类）
2. 处理 @PropertySource
   1. @PropertySource 用来加载 properties 文件
3. 获得 @ComponentScan 具体内容
4. 判断是否被 @ComponentScans 标记，或者被 @Condition 标记
   1. `是`
      1. **执行扫描操作**（重点），把扫描出来的类放入 Set
      2. 循环 Set，判断是否为配置类
         1. `是`
            1. 递归调用 `parse()`
            2. 因为被扫描出来的配置类，可能有 @ComponentScans 或者 @Bean 标注的方法，需要再次被解析
5. 处理 @Import（重要注解，SpringBoot中有大量应用）
   1. @Import 的三种情况
      1. Import 普通类
      2. Import ImportSelector
      3. Import ImportBeanDefiniitionRegister
   2. **getImports()** （重点）获得 import 内容，返回 Set
6. 处理 @ImportResource
7. 处理 @Bean，不会马上转换成 BeanDefinition，而是先转换为 Set

#### 扫描Bean的操作

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608879011589.png)

```java
 public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {

        /**
         * 扫描器
         * 在 new AnnotationConfigApplicationContext() 时，构造方法中有一句
         *      this.scanner = new ClassPathBeanDefinitionScanner(this);
         *      这个对象当时说不重要，因为只适用于开发人员手动调用 .scan 方法进行扫描
         *
         * 常规用法中，实际上执行扫描的都是这里 new 的 scanner 对象
         */
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
                componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);

        // 判断是否重写了默认的命名规则
        Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
        boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
        scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
                BeanUtils.instantiateClass(generatorClass));

        ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
        if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
            scanner.setScopedProxyMode(scopedProxyMode);
        }
        else {
            Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
            scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
        }

        scanner.setResourcePattern(componentScan.getString("resourcePattern"));

        /**
         * addIncludeFilter() & addExcludeFilter()，最终都是往 List<TypeFilter> 里 填充数据
         *
         * TypeFilter 是一个函数式接口
         *      在调用 scanner.addIncludeFilter()、scanner.addExceludeFilter() 时
         *      仅仅把 接口函数 塞进去了，并没有真正执行 逻辑
         */

        // 处理includeFilters
        for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
            for (TypeFilter typeFilter : typeFiltersFor(filter)) {
                scanner.addIncludeFilter(typeFilter);
            }
        }

        // 处理excludeFilters
        for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
            for (TypeFilter typeFilter : typeFiltersFor(filter)) {
                scanner.addExcludeFilter(typeFilter);
            }
        }

        boolean lazyInit = componentScan.getBoolean("lazyInit");
        if (lazyInit) {
            scanner.getBeanDefinitionDefaults().setLazyInit(true);
        }

        Set<String> basePackages = new LinkedHashSet<>();
        String[] basePackagesArray = componentScan.getStringArray("basePackages");
        for (String pkg : basePackagesArray) {
            String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
                    ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
            Collections.addAll(basePackages, tokenized);
        }

        /**
         * 从下面代码看出, @ComponentScans 指定扫描目标，除了最常用的 basePackages，还有两个方式:
         *      1. 指定 basePackageClasses
         *          即: 指定多个类，只要是与 这几个类 同级、或者是 子级，都会被扫描到，是 Spring 比较推荐的
         *
         *      2. 直接不指定，默认会把与配置类同级、或者在配置类子级的，作为 扫描目标
         */
        for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
            basePackages.add(ClassUtils.getPackageName(clazz));
        }
        if (basePackages.isEmpty()) {
            basePackages.add(ClassUtils.getPackageName(declaringClass));
        }
        
        /**
         * 把规则填充至 排除规则（List<TypeFilter>）
         *      这里就把 注册类自身当做排除规则，真正执行匹配的时候，会把自身给排除
         */
        scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
            @Override
            protected boolean matchClassName(String className) {
                return declaringClass.equals(className);
            }
        });
        // basePackages是一个LinkedHashSet<String>，这里就是把basePackages转为字符串数组的形式
        return scanner.doScan(StringUtils.toStringArray(basePackages));
    }
```

1. 初始化一个扫描器 scanner 
2. 处理 includeFilters，将 规则 添加至 scanner
3. 处理 excludeFilters，将 规则 添加至 scanner
4. 解析 basePackages，获得需要扫描哪些包
5. 添加一个默认的排除规则：排除自身
6. **执行扫描**（重点）

#### 执行扫描

```java
 protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        
        // 循环处理 basePackages
        for (String basePackage : basePackages) {
            
            // 根据包名找到符合条件的 BeanDefinition 集合
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            
            for (BeanDefinition candidate : candidates) {
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);

                /**
                 * 从 findCandidateComponents() 内部方法得知，这里 candidate 是 ScannedGenericBeanDefinition
                 * 
                 * 因为类的实现关系 -> 下面两个 if 都会进入
                 */
                if (candidate instanceof AbstractBeanDefinition) {
                    // 内部会设置默认值
                    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
                }
                if (candidate instanceof AnnotatedBeanDefinition) {
                    // 如果是AnnotatedBeanDefinition，还会再设置一次值
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
                }
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder =
                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
    }
```

- 因为 basePackages 可能有多个，所以需要循环处理，最终会进行 Bean 的注册

#### findCandidateComponents()

```java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {
        /**
         * Spring 支持 component 索引技术，需要引入一个组件
         *      而大部分情况下，不会引入这个组件，所以会进入 else 的逻辑
         */
        if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
            return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
        }
        else {
            return scanCandidateComponents(basePackage);
        }
    }
```

```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
        Set<BeanDefinition> candidates = new LinkedHashSet<>();
        try {

            /**
             * 将入参, 类似: 命名空间形式的字符串 -> 类似类文件地址的形式，然后在前面再加上 classpath*:
             *
             * 即: com.xx -> classpath*:com/xx/*.class
             */
            String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                    resolveBasePackage(basePackage) + '/' + this.resourcePattern;

            // 根据 packageSearchPath，获得符合要求的文件
            Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
            boolean traceEnabled = logger.isTraceEnabled();
            boolean debugEnabled = logger.isDebugEnabled();

            // 循环资源
            for (Resource resource : resources) {
                if (traceEnabled) {
                    logger.trace("Scanning " + resource);
                }

                // 判断资源是否可读，并且不是一个目录
                if (resource.isReadable()) {
                    try {
                        // metadataReader 元数据读取器，解析resource，也可以理解为描述资源的数据结构
                        MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
     
                        /**
                         * 在 isCandidateComponent() 内部，会真正执行 匹配规则
                         * 注册配置类自身会被排除，不会进入该 if
                         */
                        if (isCandidateComponent(metadataReader)) {
                            ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                            sbd.setResource(resource);
                            sbd.setSource(resource);
                            if (isCandidateComponent(sbd)) {
                                if (debugEnabled) {
                                    logger.debug("Identified candidate component class: " + resource);
                                }
                                candidates.add(sbd);
                            }
                            else {
                                if (debugEnabled) {
                                    logger.debug("Ignored because not a concrete top-level class: " + resource);
                                }
                            }
                        }
                        else {
                            if (traceEnabled) {
                                logger.trace("Ignored because not matching any filter: " + resource);
                            }
                        }
                    }
                    catch (Throwable ex) {
                        throw new BeanDefinitionStoreException(
                                "Failed to read candidate component class: " + resource, ex);
                    }
                }
                else {
                    if (traceEnabled) {
                        logger.trace("Ignored because not readable: " + resource);
                    }
                }
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
        }
        return candidates;
    }
```

1. 文件描述的转换, 获得 `packageSearchPath`
   1. com.xx -> classpath*:com/xx/`*`.class
2. 根据 `packageSearchPath`，获得符号要求的文件
3. 遍历 符合要求的文件，进一步判断
4. 最终把 符合要求的文件，转换为 BeanDefinition，并返回

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608883039964.png)

- 到这里，invokeBeanFactoryPostprocessor() **执行扫描操作** 算是完事了

#### @Import 的解析

```java
/**
     * 这个方法内部相当复杂...
     *
     * importCandidates 是 Import 的具体内容，调用该方法时，进行遍历，会有三种情况:
     *      1. Import 普通类
     *          进入 else 逻辑，到调用 processConfigurationClass()，（ processImports() 就是 在 processConfigurationClass() 方法中调用的 ）
     *          这里是一个 递归调用，因为 Import 普通类，也有可能被加了 @Import、@ComponentScan、...
     *          所系需要再次解析
     *
     *      2. Import ImportSelector
     *          进入第一个 if 逻辑，首先会执行 Aware 的接口方法，所以开发在实现 ImportSelector 的同时，还可以实现 Aware 接口
     *          接下来判断是不是 DeferredImportSelector （ 一个扩展了 ImportSelector 的类 ）
     *              不是:
     *                  调用 selectImports()，获得全限定类名数组，再转换成 类数组，再调用 processImports()...
     *                  又是一个 递归调用...
     *
     *      3. Import ImportBeanDefinitionRegistrar
     *          进入第二个 if 逻辑，仍然执行 Aware 接口方法
     *          然后把数据添加至 ConfigurationClass 中的 Map importBeanDefinitionRegistrars 中
     *          这里没有递归了..
     */
    private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
                                Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

        if (importCandidates.isEmpty()) {
            return;
        }

        if (checkForCircularImports && isChainedImportOnStack(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
        } else {
            this.importStack.push(configClass);
            try {
                for (SourceClass candidate : importCandidates) {
                    if (candidate.isAssignable(ImportSelector.class)) {
                        // Candidate class is an ImportSelector -> delegate to it to determine imports
                        Class<?> candidateClass = candidate.loadClass();
                        ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                        ParserStrategyUtils.invokeAwareMethods(
                                selector, this.environment, this.resourceLoader, this.registry);
                        if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                            this.deferredImportSelectors.add(
                                    new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                        } else {
                            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                            processImports(configClass, currentSourceClass, importSourceClasses, false);
                        }
                    } else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                        // Candidate class is an ImportBeanDefinitionRegistrar ->
                        // delegate to it to register additional bean definitions
                        Class<?> candidateClass = candidate.loadClass();
                        ImportBeanDefinitionRegistrar registrar =
                                BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                        ParserStrategyUtils.invokeAwareMethods(
                                registrar, this.environment, this.resourceLoader, this.registry);
                        configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                    } else {
                        // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                        // process it as an @Configuration class
                        this.importStack.registerImport(
                                currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                        processConfigurationClass(candidate.asConfigClass(configClass));
                    }
                }
            } catch (BeanDefinitionStoreException ex) {
                throw ex;
            } catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to process import candidates for configuration class [" +
                                configClass.getMetadata().getClassName() + "]", ex);
            } finally {
                this.importStack.pop();
            }
        }
    }
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608884141430.png)

- 到这里，算是把 ConfigurationClassPostProcessor.processConfigBeanDefiniition() 简单过了一遍
- 但是到这里还没有结束，这里只会解析 @Import 的 Bean 而已，并不会注册
- 后续再执行 ConfigurationClassPostProcessor 扩展 BeanFactoryPostProcessor 的方法
  - `postProcessorBeanFactory()`：
    - 判断 配置类 
      - Full配置类: 就会被 Cglib 代理
      - Lite配置类: 返回自身 bean

#### 小结

- ConfigurationClassPostProcessor.processConfigBeanDefiniition() 
  - 主要完成 扫描，并最终 注册 成我们定义的 Bean

### registerBeanPostProcessors(beanFactory)

- 实例化 & 注册 beanFactory 中扩展了 `BeanPostProcessor` 的 bean
- 例如:
  - `AutowiredAnnotationBeanPostProcessor`：处理被 @Autowired 修饰的 bean 并注入
  - `RequiredAnnotationBeanPostProcessor`：处理被 @Requuired 修饰的方法
  - `CommonAnnotationBeanPostProcessor`：处理 @PreDestroy、@PostConstruct、@Resource...

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608885887701.png)

### finishBeanFactoryInitialization(beanFactory)

- 实例化所有与剩余的 （非懒加载）单例bean
- 例如: 
  - `invokeBeanFactoryPostProcessors()` 根据各种注解解析出来的类，在这个时候都会被 `初始化`
- 实例化过程中，各种 `BeanPostProcessor` 开始起作用

#### beanFactory.preInstantiateSingletons()

```java
/**
 * finishBeanFactoryInitialization() 内部方法
 * 初始化所有的 非懒加载 单例 bean
 */
beanFactory.preInstantiateSingletons()
```

#### DefaultListableBeanFactory.getBean()

```java
/**
 * 上述点进来的接口 具体实现类 调用方法
 * 在循环中调用 getBean()
 */
getBean(beanName);
```

#### 门面 doGetBean()

```java
/**
 * 这里有个分支:
 * 		Bean 是 FactoryBean
 *		
 *		Bean 不是 FactoryBean
 *
 * 但最终都会进入 doGetBean()
 */
return doGetBean(name, null, null, false);
```

#### 核心代码

```java
if (mbd.isSingleton()) {
        /**
         * getSingleton() 第二个参数是一个 函数接口Function
         *      不会立即执行，而是在 方法内部中，调用 ObjectFactory.getObject() 时执行
         *      
         *      即: 真正的 createBean()
         */
        sharedInstance = getSingleton(beanName, () -> {
            try {
                return createBean(beanName, mbd, args);
            }
            catch (BeansException ex) {
                destroySingleton(beanName);
                throw ex;
            }
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
    }
```

#### AbstractAutowireCapableBeanFactory.doCreateBean()

```java
/**
 * createBean() 最终调到的门面方法
 */

// 创建bean，核心
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
return beanInstance;
```

#### 创建实例

```java
/**
 * 继续深入，就到了 Bean 生命周期的部分了
 */

// 创建bean的实例。核心
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

#### 填充属性

```java
// 填充属性，超级重要
populateBean(beanName, mbd, instanceWrapper);
```

#### Aware 系列接口的回调

```java
/**
 * 填充属性下面有一行代码:
 * 		exposedObject = initializeBean(beanName, exposedObject, mbd);
 *
 * 继续深入该方法
 * 		Aware 系列接口的回调位于 initializeBean() 中的 invokeAwareMethods()
 */
private void invokeAwareMethods(final String beanName, final Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ClassLoader bcl = getBeanClassLoader();
                if (bcl != null) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                }
            }
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
        }
    }
```

#### BeanPostProcessor.postProcessBeforeInitialization()

```java
/**
 * 该方法同样在 initializeBean() 中调用
 */
if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
        }
    @Override
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
            throws BeansException {

        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessBeforeInitialization(result, beanName);
            if (current == null) {
                return result;
            }
            result = current;
        }
        return result;
    }
```

#### afterPropertiesSet & init-method

```java
/**
 * 该方法同样在 initializeBean() 中调用
 */
invokeInitMethod(beaenName, wrappedBean, mbd);

/**
 * 又调用以下两个方法:
 * 		afterPropertiesSet()
 * 		init-method()
 */
((InitializingBean) bean).afterPropertiesSet();
invokeCustomInitMethod(beanName, bean, mbd);
```

#### BeanPostProcessor.postProcessAfterInitialization()

```java
/**
 * 该方法同样在 initializeBean() 中调用
 */
if (mbd == null || !mbd.isSynthetic()) {
            wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
    public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
            throws BeansException {

        Object result = existingBean;
        for (BeanPostProcessor processor : getBeanPostProcessors()) {
            Object current = processor.postProcessAfterI nitialization(result, beanName);
            if (current == null) {
                return result;
            }
            result = current;
        }
        return result;
    }
```

- 实际开发中，应该没人手动销毁 Spring Context上下文，剩下的两个 destory()  回调就不找了



## Spring Bean 的生命周期描述

1. 实例化Bean
2. 填充属性
3. 判断是否实现 Aware 接口、是则进行回调
   1. `BeanNameAware`: 调用 setBeanName()
   2. `BeanClassLoaderAware`: 调用 setBeanClassLoader()
   3. `BeanFactoryAware`: 调用 setBeanFactory()
4. 调用 BeanPostProcessor.postProcessBeforeInitialization()
5. 判断是否实现 InitialzingBean 接口
   1. `是`：调用 afterPropertiesSet()
6. 判断是否定义 init-method()
   1. `是`：调用 init-method()
7. 调用 BeanPostProcessor.postProcessAfterInitialization()
   1. 到这里，Bean 就准备完毕，一直停留在 Spring Context上下文，直到被销毁
8. Spring Context上下文销毁
   1. 判断是否实现 DisposableBean 接口
      1. `是`：调用 destroy()
   2. 判断是否定义 destory-method()
      1. `是`: 调用 destroy-method()