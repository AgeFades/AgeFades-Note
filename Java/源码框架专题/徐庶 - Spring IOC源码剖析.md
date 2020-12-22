# 徐庶 - Spring IOC源码剖析

## IOC容器加载过程源码

### AnnotationConfigApplicationContext

- 基于注解配置的 应用上下文、设计理念要比 xml 类的上下文更先进

### 类结构图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608628685299.png)

### 源码流程

1. 调用有参构造函数，创建Spring应用上下文，即 IOC 容器

```java
/**
 * 实际调用的有参构造函数
 * 会先调用无参构造函数
 */
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
  // 调用无参构造函数
  this();
  // 注册自己的配置类
  register(componentClasses);
  // IOC容器进行刷新的接口
  refresh();
}
```

2. 有参构造调用无参构造

```java
/**
 * 无参构造函数
 * 子类调用无参构造函数，会先调用父类无参构造函数
 */
public AnnotationConfigApplicationContext() {
  
  // 创建一个读取 注解的Bean定义 读取器
  // Bean定义 -> BeanDefinition
  // 完成了Spring内部BeanDefinition的注册（主要是后置处理器）
  this.reader = new AnnotatedBeanDefinitionReader(this);
  
  // 创建 BeanDefinition 扫描器
  // 可以用来扫描 包或者类，继而转换为 BeanDefinition
  this.scanner = new ClassPathBeanDefinitionScanner(this);
  
}
```

3. 无参构造调用父类无参构造

```java
public GenericApplicationContext() {
  
   // 为Spring应用上下文 ApplicationContext 初始 BeanFactory
   // 实现类是 DefaultListableBeanFactory 的原因是:
   // DefaultListableBeanFactory 是最低层的实现，功能最全
   this.beanFactory = new DefaultListableBeanFactory();
  
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608629239769.png)

4. new AnnotatedBeanDefinitionReader(this) 做的事

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
  this(registry, getOrCreateEnvironment(registry));
}

public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
  
  Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
  Assert.notNull(environment, "Environment must not be null");
  
  // 把 ApplicationContext 赋值给 AnnotatedBeanDefinitionReader
  this.registry = registry;
  
  // 用户处理条件注解 @Conditional os.name
  this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
  
  // 注册一些内置的后置处理器
  AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
  
}
```

5. 注册内置后置处理器

```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				//注册了实现Order接口的排序器
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			//设置@AutoWired的候选的解析器
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

		/**
		 * 为我们容器中注册了解析我们配置类的后置处理器ConfigurationClassPostProcessor
		 * 名字叫:org.springframework.context.annotation.internalConfigurationAnnotationProcessor
		 */
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 为我们容器中注册了处理@Autowired 注解的处理器AutowiredAnnotationBeanPostProcessor
		 * 名字叫:org.springframework.context.annotation.internalAutowiredAnnotationProcessor
		 */
		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 为我们容器中注册处理@Required属性的注解处理器RequiredAnnotationBeanPostProcessor
		 * 名字叫:org.springframework.context.annotation.internalRequiredAnnotationProcessor
		 */
		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 为我们容器注册处理JSR规范的注解处理器CommonAnnotationBeanPostProcessor
		 * org.springframework.context.annotation.internalCommonAnnotationProcessor
		 */
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 处理jpa注解的处理器org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor
		 */
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 处理监听方法的注解解析器EventListenerMethodProcessor
		 */
		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}

		/**
		 * 注册事件监听器工厂
		 */
		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```





## BeanFactory 和 FactoryBean 的区别

## Bean的生命周期源码

## 初识源码中的各种扩展点

## BeanDefinition详解

