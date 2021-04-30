[TOC]

# 徐庶 - Spring IOC源码剖析

## IOC容器加载过程源码

### AnnotationConfigApplicationContext

- 基于注解配置的 应用上下文、设计理念要比 xml 类的上下文更先进

### 类结构图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608628685299.png)

### 源码流程

#### 1. 调用有参构造函数，创建Spring应用上下文，即 IOC 容器

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

#### 2. 有参构造调用无参构造

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
  
  // 创建 ClassPath 类型的 BeanDefinition 扫描器
  // 可以用来扫描 包或者类，继而转换为 BeanDefinition
  this.scanner = new ClassPathBeanDefinitionScanner(this);
  
}
```

#### 3. 无参构造调用父类无参构造

```java
public GenericApplicationContext() {
  
   // 为Spring应用上下文 ApplicationContext 初始 BeanFactory
   // 实现类是 DefaultListableBeanFactory 的原因是:
   // DefaultListableBeanFactory 是最低层的实现，功能最全
   this.beanFactory = new DefaultListableBeanFactory();
  
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608629239769.png)

#### 4. new AnnotatedBeanDefinitionReader(this) 做的事

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

#### 5. 注册内置后置处理器

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

#### 6. ClassPathBeanDefinitionScanner 的核心函数

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
  
		Assert.notEmpty(basePackages, "At least one base package must be specified");
  
		// 创建bean定义的holder对象用于保存扫描后生成的bean定义对象
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
  
		// 循环我们的包路径集合
		for (String basePackage : basePackages) {
      
			// 找到候选的Compents
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			for (BeanDefinition candidate : candidates) {

				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				candidate.setScope(scopeMetadata.getScopeName());
        
				// 设置我们的beanName
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
        
				// 处理@AutoWired相关的
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
        
				// 处理jsr250相关的组件
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
        
				// 把我们解析出来的组件bean定义注册到我们的IOC容器中
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

#### 7. 最重要的 refresh() 方法 -> 实际调用父类AbstractApplicationContext 的 refresh() 实现

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    //1:准备刷新上下文环境
    prepareRefresh();

    //2:获取告诉子类初始化Bean工厂
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    //3:对bean工厂进行填充属性
    prepareBeanFactory(beanFactory);

    try {
      // 第四:留个子类去实现该接口
      postProcessBeanFactory(beanFactory);

      // 调用我们的bean工厂的后置处理器.
      invokeBeanFactoryPostProcessors(beanFactory);

      // 调用我们bean的后置处理器
      registerBeanPostProcessors(beanFactory);

      // 初始化国际化资源处理器.
      initMessageSource();

      // 创建事件多播器
      initApplicationEventMulticaster();

      // 这个方法同样也是留个子类实现的springboot也是从这个方法进行启动tomat的.
      onRefresh();

      //把我们的事件监听器注册到多播器上
      registerListeners();

      //实例化我们剩余的单实例bean.
      finishBeanFactoryInitialization(beanFactory);

      // 最后容器刷新 发布刷新事件(Spring cloud也是从这里启动的)
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception  encountered during context initialization - " +
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

#### 8. IOC相关之 Bean工厂后置处理器的调用

```java
/**
 * 这里实际调用了之前创建的 BeanDefinitionReader 构造中、
 * 创建的各种 IOC 容器不可或缺的 BeanFactoryPostProcessor 的实现方法
 *
 * 在 BeanDefinitionReader 创建时，已经将这些 Bean工厂后置处理器 转成 RootBeanDefinition
 * 并注册至 BeanDefinitionMap，所以这里 invoke 实质上是 getBean 实例化这些后置处理器并调用
 *
 * 比如最重要的: ConfigurationClassPostProcessor
 * ConfigurationClassPostProcessor 可以将 Reader 读取中的 Java 类
 * 相应解析成 BeanDefinition，并注册到 BeanFacotry 中的 BeanDefinitionMap
 */
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		//传入bean工厂和获取applicationContext中的bean工厂后置处理器(但是由于没有任何实例化过程,所以传递进来的为空)
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

#### 9. invokeBeanFactoryPostProcessors() 调用链中的 ConfigurationClassPostProcessor.processConfigBeanDefinitions()

```java
/**
 * 解析配置类成 BeanDefinition
 */
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		//获取IOC 容器中目前所有bean定义的名称
		String[] candidateNames = registry.getBeanDefinitionNames();

		//循环我们的上一步获取的所有的bean定义信息
		for (String beanName : candidateNames) {
			//通过bean的名称来获取我们的bean定义对象
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			//判断是否有没有解析过
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
					ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			//进行正在的解析判断是不是完全的配置类 还是一个非正式的配置类
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				//满足添加 就加入到候选的配置类集合中
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}

		// 若没有找到配置类 直接返回
		if (configCandidates.isEmpty()) {
			return;
		}

		//对我们的配置类进行Order排序
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});

		// 创建我们通过@CompentScan导入进来的bean name的生成器
		// 创建我们通过@Import导入进来的bean的名称
		SingletonBeanRegistry sbr = null;
		if (registry instanceof SingletonBeanRegistry) {
			sbr = (SingletonBeanRegistry) registry;
			if (!this.localBeanNameGeneratorSet) {
				BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
				if (generator != null) {
					//设置@CompentScan导入进来的bean的名称生成器
					this.componentScanBeanNameGenerator = generator;
					//设置@Import导入进来的bean的名称生成器
					this.importBeanNameGenerator = generator;
				}
			}
		}

		if (this.environment == null) {
			this.environment = new StandardEnvironment();
		}

		//创建一个配置类解析器对象
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		//创建一个集合用于保存我们的配置类BeanDefinitionHolder集合默认长度是配置类集合的长度
		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		//创建一个集合用于保存我们的已经解析的配置类，长度默认为解析出来默认的配置类的集合长度
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		//do while 会进行第一次解析
		do {
			//真正的解析我们的配置类
			parser.parse(candidates);
			parser.validate();

			//解析出来的配置类
			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			//真正的把我们解析出来的配置类注册到容器中
			this.reader.loadBeanDefinitions(configClasses);
			//加入到已经解析的集合中
			alreadyParsed.addAll(configClasses);

			candidates.clear();
			//判断我们ioc容器中的是不是>候选原始的bean定义的个数
			if (registry.getBeanDefinitionCount() > candidateNames.length) {
				//获取所有的bean定义
				String[] newCandidateNames = registry.getBeanDefinitionNames();
				//原始的老的候选的bean定义
				Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
				Set<String> alreadyParsedClasses = new HashSet<>();
				//赋值已经解析的
				for (ConfigurationClass configurationClass : alreadyParsed) {
					alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
				}

				for (String candidateName : newCandidateNames) {
					//表示当前循环的还没有被解析过
					if (!oldCandidateNames.contains(candidateName)) {
						BeanDefinition bd = registry.getBeanDefinition(candidateName);
						//判断有没有被解析过
						if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
								!alreadyParsedClasses.contains(bd.getBeanClassName())) {
							candidates.add(new BeanDefinitionHolder(bd, candidateName));
						}
					}
				}
				candidateNames = newCandidateNames;
			}
		}
		//存在没有解析过的 需要循环解析
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

#### 10. 调用完Bean工厂后置处理器方法后 -> 注册 Bean后置处理器

```java
/**
 * 给我们容器中注册了我们bean的后置处理器
 * bean的后置处理器在什么时候进行调用？在bean的各个生命周期中都会进行调用
 * @param beanFactory
 * @param applicationContext
 */
public static void registerBeanPostProcessors(
  ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

  //去容器中获取所有的BeanPostProcessor 的名称(还是bean定义)
  String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

  /**
		 * bean的后置处理器的个数 beanFactory.getBeanPostProcessorCount()成品的个数 直接保存在beanPostProcessors集合中的
		 * postProcessorNames.length  beanFactory工厂中beand定义的个数
		 * +1 在后面又马上注册了BeanPostProcessorChecker的后置处理器
		 */
  int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
  beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

  /**
		 * 按照BeanPostProcessor实现的优先级接口来分离我们的后置处理器
		 */
  //保存实现了priorityOrdered接口的
  List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
  //系统内部的
  List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
  //实现了我们ordered接口的
  List<String> orderedPostProcessorNames = new ArrayList<>();
  //实现了我们任何优先级的
  List<String> nonOrderedPostProcessorNames = new ArrayList<>();
  //循环我们的bean定义(BeanPostProcessor)
  for (String ppName : postProcessorNames) {
    //若实现了PriorityOrdered接口的
    if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
      //显示的调用getBean流程创建bean的后置处理器
      BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
      //加入到集合中
      priorityOrderedPostProcessors.add(pp);
      //判断是否实现了MergedBeanDefinitionPostProcessor
      if (pp instanceof MergedBeanDefinitionPostProcessor) {
        //加入到集合中
        internalPostProcessors.add(pp);
      }
    }
    //判断是否实现了Ordered
    else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
      orderedPostProcessorNames.add(ppName);
    }
    //没有任何拍下接口的
    else {
      nonOrderedPostProcessorNames.add(ppName);
    }
  }

  // 把实现了priorityOrdered注册到容器中
  sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

  // 处理实现Ordered的bean定义
  List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
  for (String ppName : orderedPostProcessorNames) {
    //显示调用getBean方法
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    //加入到集合中
    orderedPostProcessors.add(pp);
    //判断是否实现了MergedBeanDefinitionPostProcessor
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      //加入到集合中
      internalPostProcessors.add(pp);
    }
  }
  //排序并且注册我们实现了Order接口的后置处理器
  sortPostProcessors(orderedPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, orderedPostProcessors);

  // 实例化我们所有的非排序接口的
  List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
  for (String ppName : nonOrderedPostProcessorNames) {
    //显示调用
    BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
    nonOrderedPostProcessors.add(pp);
    //判断是否实现了MergedBeanDefinitionPostProcessor
    if (pp instanceof MergedBeanDefinitionPostProcessor) {
      internalPostProcessors.add(pp);
    }
  }
  //注册我们普通的没有实现任何排序接口的
  registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

  //注册.MergedBeanDefinitionPostProcessor类型的后置处理器 bean 合并后的处理， Autowired 注解正是通过此方法实现诸如类型的预解析。
  sortPostProcessors(internalPostProcessors, beanFactory);
  registerBeanPostProcessors(beanFactory, internalPostProcessors);

  //注册ApplicationListenerDetector 应用监听器探测器的后置处理器
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

#### 11. 注册最终可用的 Bean

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// 为我们的bean工厂创建类型转化器  Convert
		/**
		 * public class String2DateConversionService implements Converter<String,Date> {

		 	public Date convert(String source) {
				SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
					try {
						return sdf.parse(source);
					} catch (ParseException e) {
					return null;
				}
		 	}
		 }

		 @Bean
		 public ConversionServiceFactoryBean conversionService() {
		 	ConversionServiceFactoryBean factoryBean = new ConversionServiceFactoryBean();
		 	Set<Converter> converterSet = new HashSet<Converter>();
		 	converterSet.add(new String2DateConversionService());
		 	factoryBean.setConverters(converterSet);
		 	return factoryBean;
		 }
		 ConversionServiceFactoryBean conversionServiceFactoryBean = (ConversionServiceFactoryBean) ctx.getBean(ConversionServiceFactoryBean.class);
		 conversionServiceFactoryBean.getObject().convert("2019-06-03 12:00:00",Date.class)
		 */
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		/**
		 * public class MainConfig implements EmbeddedValueResolverAware{
		 *
		 *     public void setEmbeddedValueResolver(StringValueResolver resolver) {
		 		this.jdbcUrl = resolver.resolveStringValue("${ds.jdbcUrl}");
		 		this.classDriver = resolver.resolveStringValue("${ds.classDriver}");
		      }
		 }
		 */
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// 处理关于aspectj
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		//冻结所有的 bean 定义 ， 说明注册的 bean 定义将不被修改或任何进一步的处理
		beanFactory.freezeConfiguration();

		//实例化剩余的单实例bean
		beanFactory.preInstantiateSingletons();
	}
```

#### 12. 实例化剩余的单实例bean -> DefaultListableBeanFactory.preInstantiateSingletons()

```java
@Override
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isDebugEnabled()) {
			logger.debug("Pre-instantiating singletons in " + this);
		}

		//获取我们容器中所有bean定义的名称
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		//循环我们所有的bean定义名称
		for (String beanName : beanNames) {
			//合并我们的bean定义
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			/**
			 * 根据bean定义判断是不是抽象的&& 不是单例的 &&不是懒加载的
			 */
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				//是不是工厂bean
				if (isFactoryBean(beanName)) {
					//是的话 给beanName+前缀&符号
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						//调用真正的getBean的流程
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {//非工厂Bean 就是普通的bean
					getBean(beanName);
				}
			}
		}

		//或有的bean的名称 ...........到这里所有的单实例的bean已经记载到单实例bean到缓存中
		for (String beanName : beanNames) {
			//从单例缓存池中获取所有的对象
			Object singletonInstance = getSingleton(beanName);
			//判断当前的bean是否实现了SmartInitializingSingleton接口
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					//触发实例化之后的方法afterSingletonsInstantiated
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

#### 13. getBean() 实际调用 -> AbstractBeanFactory.doGetBean()

```java
/**
	 * 返回bean的实例,该实例可能是单例bean 也有可能是原型的bean
	 * @param name bean的名称 也有可能是bean的别名
	 * @param requiredType 需要获取bean的类型
	 * @param args 通过该参数传递进来,到调用构造方法时候发现有多个构造方法,我们就可以通过该参数来指定想要的构造方法了
	 *             不需要去推断构造方法(因为推断构造方法很耗时)
	 * @param typeCheckOnly 判断当前的bean是不是一个检查类型的bean 这类型用的很少.
	 * @return 返回一个bean实例
	 * @throws BeansException 如何bean不能被创建 那么就回抛出异常
	 */
	@SuppressWarnings("unchecked")
	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		/**
		 * 在这里 传入进来的name 可能是 别名, 也有可能是工厂bean的name,所以在这里需要转换
		 */
		final String beanName = transformedBeanName(name);
		Object bean;

		//尝试去缓存中获取对象
		Object sharedInstance = getSingleton(beanName);

		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			/**
			 * /*
			 * 如果 sharedInstance 是普通的单例 bean，下面的方法会直接返回。但如果
			 * sharedInstance 是 FactoryBean 类型的，则需调用 getObject 工厂方法获取真正的
			 * bean 实例。如果用户想获取 FactoryBean 本身，这里也不会做特别的处理，直接返回
			 * 即可。毕竟 FactoryBean 的实现类本身也是一种 bean，只不过具有一点特殊的功能而已。
			 */
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {

			/**
			 * spring 只能解决单例对象的setter 注入的循环依赖,不能解决构造器注入
			 */
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			/**
			 * 判断AbstractBeanFacotry工厂是否有父工厂(一般情况下是没有父工厂因为abstractBeanFactory直接是抽象类,不存在父工厂)
			 * 一般情况下,只有Spring 和SpringMvc整合的时才会有父子容器的概念,
			 * 比如我们的Controller中注入Service的时候，发现我们依赖的是一个引用对象，那么他就会调用getBean去把service找出来
			 * 但是当前所在的容器是web子容器，那么就会在这里的 先去父容器找
			 */
			BeanFactory parentBeanFactory = getParentBeanFactory();
			//若存在父工厂,切当前的bean工厂不存在当前的bean定义,那么bean定义是存在于父beanFacotry中
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				//获取bean的原始名称
				String nameToLookup = originalBeanName(name);
				//若为 AbstractBeanFactory 类型，委托父类处理
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					//  委托给构造函数 getBean() 处理
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// 没有 args，委托给标准的 getBean() 处理
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			/**
			 * 方法参数 typeCheckOnly ，是用来判断调用 #getBean(...) 方法时，表示是否为仅仅进行类型检查获取 Bean 对象
			 * 如果不是仅仅做类型检查，而是创建 Bean 对象，则需要调用 #markBeanAsCreated(String beanName) 方法，进行记录
			 */
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				/**
				 * 从容器中获取 beanName 相应的 GenericBeanDefinition 对象，并将其转换为 RootBeanDefinition 对象
				 *   <bean id="tulingParentCompent" class="com.tuling.testparentsonbean.TulingParentCompent" abstract="true">
				        <property name="tulingCompent" ref="tulingCompent"></property>
				    </bean>
				 	<bean id="tulingSonCompent" class="com.tuling.testparentsonbean.TulingSonCompent" parent="tulingParentCompent"></bean>
				 */
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				//检查当前创建的bean定义是不是抽象的bean定义
				checkMergedBeanDefinition(mbd, beanName, args);

				/**
			      *
				  * @Bean
				   public DependsA dependsA() {
						return new DependsA();
				   }

					 @Bean
					 @DependsOn(value = {"dependsA"})
					 public DependsB dependsB() {
					    return new DependsB();
					 }
				 * 处理dependsOn的依赖(这个不是我们所谓的循环依赖 而是bean创建前后的依赖)
				 */
				//依赖bean的名称
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					// <1> 若给定的依赖 bean 已经注册为依赖给定的 bean
					// 即循环依赖的情况，抛出 BeanCreationException 异常
					for (String dep : dependsOn) {
						//beanName是当前正在创建的bean,dep是正在创建的bean的依赖的bean的名称
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						//保存的是依赖 beanName 之间的映射关系：依赖 beanName - > beanName 的集合
						registerDependentBean(dep, beanName);
						try {
							//获取depentceOn的bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				//创建单例bean
				if (mbd.isSingleton()) {
					//把beanName 和一个singletonFactory 并且传入一个回调对象用于回调
					sharedInstance = getSingleton(beanName, () -> {
						try {
							//进入创建bean的逻辑
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							//创建bean的过程中发生异常,需要销毁关于当前bean的所有信息
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

#### 14. 尝试从缓存中获取 bean -> DefaultSingletonBeanRegistry.getSingleton()

```java
/** 一级缓存 这个就是我们大名鼎鼎的单例缓存池 用于保存我们所有的单实例bean */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** 三级缓存 该map用户缓存 key为 beanName  value 为ObjectFactory(包装为早期对象) */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** 二级缓存 ，用户缓存我们的key为beanName value是我们的早期对象(对象属性还没有来得及进行赋值) */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	/** Set of registered singletons, containing the bean names in registration order */
	private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

	/** 该集合用户缓存当前正在创建bean的名称 */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** 排除当前创建检查的*/
	private final Set<String> inCreationCheckExclusions =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

	/** List of suppressed Exceptions, available for associating related causes */
	@Nullable
	private Set<Exception> suppressedExceptions;

	/** Flag that indicates whether we're currently within destroySingletons */
	private boolean singletonsCurrentlyInDestruction = false;

	/** 用户缓存记录实现了DisposableBean 接口的实例 */
	private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

	/** 缓存bean的属性关系的映射<tulingServce,<tulingDao,tulingDao2> --> Set of bean names that the bean contains */
	private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

	/**
	 * Map between dependent bean names: bean name to Set of dependent bean names.
	 *
	 * 保存的是依赖 beanName 之间的映射关系：beanName - > 依赖 beanName 的集合
	 */
	private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

	/**
	 * Map between depending bean names: bean name to Set of bean names for the bean's dependencies.
	 *
	 * 保存的是依赖 beanName 之间的映射关系：依赖 beanName - > beanName 的集合
	 */
	private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);



/**
	 * 该方法是一个空壳方法
	 * @param beanName bean的名称
	 * @return 缓存中的对象(有可能是一个单例完整对象,也有可能是一个早期对象(用于解决循环依赖))
	 */
	@Override
	@Nullable
	public Object getSingleton(String beanName) {
		//在这里 系统一般是允许早期对象引用的 allowEarlyReference通过这个参数可以控制解决循环依赖
		return getSingleton(beanName, true);
	}



/**
	 * 在网上很多很多写源码的大佬或者是<spring源码深度解析>一书上,也没有说清楚为啥要使用三级缓存(二级缓存可不可以能够
	 * 解决) 答案是：可以, 但是没有很好的扩展性为啥这么说.......
	 * 原因: 获取三级缓存-----getEarlyBeanReference()经过一系列的后置处理来给我们早期对象进行特殊化处理
	 * //从三级缓存中获取包装对象的时候 ，他会经过一次后置处理器的处理对我们早期对象的bean进行
	 * 特殊化处理，但是spring的原生后置处理器没有经过处理，而是留给了我们程序员进行扩展
	 * singletonObject = singletonFactory.getObject();
	 * 把三级缓存移植到二级缓存中
	 * this.earlySingletonObjects.put(beanName, singletonObject);
	 * //删除三级缓存中的之
	 * this.singletonFactories.remove(beanName);
	 * @param beanName bean的名称
	 * @param allowEarlyReference 是否允许暴露早期对象  通过该参数可以控制是否能够解决循环依赖的.
	 * @return
	 * 	    这里可能返回一个null（IOC容器加载单实例bean的时候,第一次进来是返回null）
	 * 	    也有可能返回一个单例对象(IOC容器加载了单实例了,第二次来获取当前的Bean)
	 * 	    也可能返回一个早期对象(用于解决循环依赖问题)
	 */
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		/**
		 * 第一步:我们尝试去一级缓存(单例缓存池中去获取对象,一般情况从该map中获取的对象是直接可以使用的)
		 * IOC容器初始化加载单实例bean的时候第一次进来的时候 该map中一般返回空
		 */
		Object singletonObject = this.singletonObjects.get(beanName);
		/**
		 * 若在第一级缓存中没有获取到对象,并且singletonsCurrentlyInCreation这个list包含该beanName
		 * IOC容器初始化加载单实例bean的时候第一次进来的时候 该list中一般返回空,但是循环依赖的时候可以满足该条件
		 */
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				/**
				 * 尝试去二级缓存中获取对象(二级缓存中的对象是一个早期对象)
				 * 何为早期对象:就是bean刚刚调用了构造方法，还来不及给bean的属性进行赋值的对象
				 * 就是早期对象
				 */
				singletonObject = this.earlySingletonObjects.get(beanName);
				/**
				 * 二级缓存中也没有获取到对象,allowEarlyReference为true(参数是有上一个方法传递进来的true)
				 */
				if (singletonObject == null && allowEarlyReference) {
					/**
					 * 直接从三级缓存中获取 ObjectFactory对象 这个对接就是用来解决循环依赖的关键所在
					 * 在ioc后期的过程中,当bean调用了构造方法的时候,把早期对象包裹成一个ObjectFactory
					 * 暴露到三级缓存中
					 */
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					//从三级缓存中获取到对象不为空
					if (singletonFactory != null) {
						/**
						 * 在这里通过暴露的ObjectFactory 包装对象中,通过调用他的getObject()来获取我们的早期对象
						 * 在这个环节中会调用到 getEarlyBeanReference()来进行后置处理
						 */
						singletonObject = singletonFactory.getObject();
						//把早期对象放置在二级缓存,
						this.earlySingletonObjects.put(beanName, singletonObject);
						//ObjectFactory 包装对象从三级缓存中删除掉
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

#### 15. 第一次创建Bean、第一次调用Bean后置处理器 -> AbstractAutowireCapableBeanFactory.createBean()

```java
/**
	 * Central method of this class: creates a bean instance,
	 * populates the bean instance, applies post-processors, etc.
	 * @see #doCreateBean
	 */
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// 确保此时的 bean 已经被解析了
		// 如果获取的class 属性不为null，则克隆该 BeanDefinition
		// 主要是因为该动态解析的 class 无法保存到到共享的 BeanDefinition
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			/**
			 * 验证和准备覆盖方法
			 * lookup-method 和 replace-method
			 * 这两个配置存放在 BeanDefinition 中的 methodOverrides
			 * 我们知道在 bean 实例化的过程中如果检测到存在 methodOverrides ，
			 * 则会动态地位为当前 bean 生成代理并使用对应的拦截器为 bean 做增强处理。
			 * 具体的实现我们后续分析，现在先看 mbdToUse.prepareMethodOverrides() 代码块
			 */
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			/**
			 * 通过bean的后置处理器来进行后置处理生成代理对象,一般情况下在此处不会生成代理对象
			 * 为什么不能生成代理对象,不管是我们的jdk代理还是cglib代理都不会在此处进行代理，因为我们的
			 * 真实的对象没有生成,所以在这里不会生成代理对象，那么在这一步是我们aop和事务的关键，因为在这里
			 * 解析我们的aop切面信息进行缓存
			 */
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			/**
			 * 该步骤是我们真正的创建我们的bean的实例对象的过程
			 */
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}





	/**
	 * 真的创建bean的逻辑,该方法是最复杂的包含了调用构造函数,给bean的属性赋值
	 * 调用bean的初始化操作以及 生成代理对象 都是在这里
	 * @param beanName bean的名称
	 * @param mbd bean的定义
	 * @param args 传入的参数
	 * @return bean的实例
	 * @throws BeanCreationException if the bean could not be created
	 * @see #instantiateBean
	 * @see #instantiateUsingFactoryMethod
	 * @see #autowireConstructor
	 */
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		//BeanWrapper 是对 Bean 的包装，其接口中所定义的功能很简单包括设置获取被包装的对象，获取被包装 bean 的属性描述器
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			//从没有完成的FactoryBean中移除
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			//使用合适的实例化策略来创建新的实例：工厂方法、构造函数自动注入、简单初始化 该方法很复杂也很重要
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		//从beanWrapper中获取我们的早期对象
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					//进行后置处理 @AutoWired的注解的预解析
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		/**
		 * 该对象进行判断是否能够暴露早期对象的条件
		 * 单实例
		 * this.allowCircularReferences 默认为true
		 * isSingletonCurrentlyInCreation(表示当前的bean对象正在创建singletonsCurrentlyInCreation包含当前正在创建的bean)
		 */
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		//上述条件满足，允许中期暴露对象
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			//把我们的早期对象包装成一个singletonFactory对象 该对象提供了一个getObject方法,该方法内部调用getEarlyBeanReference方法
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//给我们的属性进行赋值(调用set方法进行赋值)
			populateBean(beanName, mbd, instanceWrapper);
			//进行对象初始化操作(在这里可能生成代理对象)
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		//允许早期对象的引用
		if (earlySingletonExposure) {
			/**
			 * 去缓存中获取到我们的对象 由于传递的allowEarlyReferce 是false 要求只能在一级二级缓存中去获取
			 * 正常普通的bean(不存在循环依赖的bean) 创建的过程中，压根不会把三级缓存提升到二级缓存中
			 */

			Object earlySingletonReference = getSingleton(beanName, false);
			//能够获取到
			if (earlySingletonReference != null) {
				//经过后置处理的bean和早期的bean引用还相等的话(表示当前的bean没有被代理过)
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				//处理依赖的bean
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			//注册销毁的bean的销毁接口
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}





@Nullable
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			//判断容器中是否有InstantiationAwareBeanPostProcessors
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				//获取当前bean 的class对象
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					/**
					 * 后置处理器的【第一次】调用 总共有九处调用 事务在这里不会被调用，aop的才会被调用
					 * 为啥aop在这里调用了，因为在此处需要解析出对应的切面报错到缓存中
					 */
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					//若InstantiationAwareBeanPostProcessors后置处理器的postProcessBeforeInstantiation返回不为null

					//说明生成了代理对象那么我们就调用
					if (bean != null) {
						/**
						 * 后置处理器的第二处调用，该后置处理器若被调用的话，那么第一处的处理器肯定返回的不是null
						 * InstantiationAwareBeanPostProcessors后置处理器postProcessAfterInitialization
						 */

						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}



@Nullable
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		/**
		 * 获取容器中的所有后置处理器
		 */
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			//判断后置处理器是不是InstantiationAwareBeanPostProcessor
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				//把我们的BeanPostProcessor强制转为InstantiationAwareBeanPostProcessor
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				/**
				 * 【很重要】
				 * 我们AOP @EnableAspectJAutoProxy 为我们容器中导入了 AnnotationAwareAspectJAutoProxyCreator
				 * 我们事务注解@EnableTransactionManagement 为我们的容器导入了 InfrastructureAdvisorAutoProxyCreator
				 * 都是实现了我们的 BeanPostProcessor接口,InstantiationAwareBeanPostProcessor,
				 * 进行后置处理解析切面
				 */
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```

#### 16. 对 Bean 的实例化 -> AbstractAutowireCapableBeanFactory.createBeanInstance()

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		//从bean定义中解析出当前bean的class对象
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		/*
		 * 检测类的访问权限。默认情况下，对于非 public 的类，是允许访问的。
		 * 若禁止访问，这里会抛出异常
     	*/
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		/**
		 * 该方法是spring5.0 新增加的  如果存在 Supplier 回调，则使用给定的回调方法初始化策略
		 */
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}

		/**
		 * 工厂方法,我们通过配置类来进行配置的话 采用的就是工厂方法,方法名称就是tulingDao就是我们工厂方法的名称
		 *  Bean
		 	public TulingDao tulingDao() {

		 		return new TulingDao(tulingDataSource());
		 	}
		 */
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

	    /**
		 * 当多次构建同一个 bean 时，可以使用此处的快捷路径，即无需再次推断应该使用哪种方式构造实例，
		 * 以提高效率。比如在多次构建同一个 prototype 类型的 bean 时，就可以走此处的捷径。
		 * 这里的 resolved 和 mbd.constructorArgumentsResolved 将会在 bean 第一次实例
		 * 化的过程中被设置，在后面的源码中会分析到，先继续往下看。
         */
		//判断当前构造函数是否被解析过
		boolean resolved = false;
		//有没有必须进行依赖注入
		boolean autowireNecessary = false;
		/**
		 * 通过getBean传入进来的构造函数是否来指定需要推断构造函数
		 * 若传递进来的args不为空，那么就可以直接选出对应的构造函数
		 */
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				//判断我们的bean定义信息中的resolvedConstructorOrFactoryMethod(用来缓存我们的已经解析的构造函数或者工厂方法)
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					//修改已经解析过的构造函数的标志
					resolved = true;
					//修改标记为ture 标识构造函数或者工厂方法已经解析过
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		//若被解析过
		if (resolved) {
			if (autowireNecessary) {
				//通过有参的构造函数进行反射调用
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				//调用无参数的构造函数进行创建对象
				return instantiateBean(beanName, mbd);
			}
		}

		/**
		 * 通过bean的后置处理器进行选举出合适的构造函数对象
		 */
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		/**
		 * 通过后置处理器解析出构造器对象不为null
		 * 获取bean定义中的注入模式是构造器注入
		 * bean定义信息ConstructorArgumentValues
		 * 获取通过getBean的方式传入的构造器函数参数类型不为null
		 */

		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			//通过构造函数创建对象
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		//使用无参数的构造函数调用创建对象
		return instantiateBean(beanName, mbd);
	}
```



#### 17. 对 Bean 的属性赋值 -> AbstractAutowireCapableBeanFactory.populateBean()

```java
/**
	 *给我们的对象BeanWrapper赋值
	 * @param beanName bean的名称
	 * @param mbd bean的定义
	 * @param bw bean实例包装对象
	 */
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		//若bw为null的话,则说明对象没有实例化
		if (bw == null) {
			//进入if 说明对象有属性，bw为空,不能为他设置属性，那就在下面就执行抛出异常
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		/**
         * 在属性被填充前，给 InstantiationAwareBeanPostProcessor 类型的后置处理器一个修改
         * bean 状态的机会。官方的解释是：让用户可以自定义属性注入。比如用户实现一
         * 个 InstantiationAwareBeanPostProcessor 类型的后置处理器，并通过
         * postProcessAfterInstantiation 方法向 bean 的成员变量注入自定义的信息。
         *当时我们发现系统中的的InstantiationAwareBeanPostProcessor.postProcessAfterInstantiationM没有进行任何处理，
         *若我们自己实现了这个接口 可以自定义处理.....spring 留给我们自己扩展接口的
         *特殊需求，直接使用配置中的信息注入即可。
         */
		boolean continueWithPropertyPopulation = true;
		//是否持有 InstantiationAwareBeanPostProcessor
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			//获取容器中的所有的BeanPostProcessor
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				//判断我们的后置处理器是不是InstantiationAwareBeanPostProcessor
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					//进行强制转化
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					//若存在后置处理器给我们属性赋值了，那么返回false 可以来修改我们的开关变量，就不会走下面的逻辑了
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						// 返回值为是否继续填充 bean
						// postProcessAfterInstantiation：如果应该在 bean上面设置属性则返回 true，否则返回 false
						// 一般情况下，应该是返回true 。
						// 返回 false 的话，将会阻止在此 Bean 实例上调用任何后续的 InstantiationAwareBeanPostProcessor 实
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}
		// 如果后续处理器发出停止填充命令，则终止后续操作
		if (!continueWithPropertyPopulation) {
			return;
		}

		//获取bean定义的属性
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

		/**
		 * 判断我们的bean的属性注入模型
		 * AUTOWIRE_BY_NAME 根据名称注入
		 * AUTOWIRE_BY_TYPE 根据类型注入
		 */

		if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
			//把PropertyValues封装成为MutablePropertyValues
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			//根据bean的属性名称注入
			if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			//根据bean的类型进行注入
			if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			//把处理过的 属性覆盖原来的
			pvs = newPvs;
		}

		/**
         * 这里又是一种后置处理，用于在 Spring 填充属性到 bean 对象前，对属性的值进行相应的处理，
         * 比如可以修改某些属性的值。这时注入到 bean 中的值就不是配置文件中的内容了，
         * 而是经过后置处理器修改后的内容
         */
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		//判断是否需要检查依赖
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			//提出当前正在创建的beanWrapper 依赖的对象
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				//获取所有的后置处理器
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						//对依赖对象进行后置处理
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			//判断是否检查依赖
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		/**
		 * 其实，上面只是完成了所有注入属性的获取，将获取的属性封装在 PropertyValues 的实例对象 pvs 中，
		 * 并没有应用到已经实例化的 bean 中。而 #applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) 方法，
		 * 则是完成这一步骤的
		 */
		if (pvs != null) {
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}



	protected void autowireByName(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
		//从我们的bean定义中解析出我们的所有属性对象
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		//循环我们当前bean的属性
		for (String propertyName : propertyNames) {
			//判断该属性名称是不是引用对象
			if (containsBean(propertyName)) {
				//显示的调用getBean所有属性的名称bean显示调用BeanFactory
				Object bean = getBean(propertyName);
				//把我们依赖的属性添加到pvs中
				pvs.add(propertyName, bean);
				//注册当前bean和属性依赖bean的依赖关系
				registerDependentBean(propertyName, beanName);
				if (logger.isDebugEnabled()) {
					logger.debug("Added autowiring by name from bean name '" + beanName +
							"' via property '" + propertyName + "' to bean named '" + propertyName + "'");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
							"' by name: no matching bean found");
				}
			}
		}
	}




	protected void autowireByType(
			String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

		// 获取 TypeConverter 实例
		// 使用自定义的 TypeConverter，用于取代默认的 PropertyEditor 机制
		TypeConverter converter = getCustomTypeConverter();
		//没有获取到  把bw赋值给转换器(BeanWrapper实现了 TypeConverter接口 )
		if (converter == null) {
			converter = bw;
		}

		Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
		/**
		 *   spring认为的简单属性
		 *   1. CharSequence 接口的实现类，比如 String
		 *   2. Enum
		 *   3. Date
		 *   4. URI/URL
		 *   5. Number 的继承类，比如 Integer/Long
		 *   6. byte/short/int... 等基本类型
		 *   7. Locale
		 *   8. 以上所有类型的数组形式，比如 String[]、Date[]、int[] 等等
		 * 不包含在当前bean的配置文件的中属性 !pvs.contains(pd.getName()
		 * */
		String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
		//循环属性,循环我们的依赖属性
		for (String propertyName : propertyNames) {
			try {
				//获取 PropertyDescriptor 实例
				PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
				//判断当前正在创建的bean中所依赖的属性描述的类型不是为object类型的
				if (Object.class != pd.getPropertyType()) {
					//获取当前正常创建备案所依赖的属性的setter方法对象
					MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
					// Do not allow eager init for type matching in case of a prioritized post-processor.
					boolean eager = !PriorityOrdered.class.isInstance(bw.getWrappedInstance());
					//创建自动注入的依赖描述符创建
					DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
					// 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称存储在 autowiredBeanNames 中
					// 当属性存在过个封装 bean 时将会找到所有匹配的 bean 并将其注入
					Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
					//若解析出来的不为空autowiredArgument  把属性名称和对应的值进绑定
					if (autowiredArgument != null) {
						pvs.add(propertyName, autowiredArgument);
					}
					for (String autowiredBeanName : autowiredBeanNames) {
						//注册依赖
						registerDependentBean(autowiredBeanName, beanName);
						if (logger.isDebugEnabled()) {
							logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
									propertyName + "' to bean named '" + autowiredBeanName + "'");
						}
					}
					autowiredBeanNames.clear();
				}
			}
			catch (BeansException ex) {
				throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
			}
		}
	}
```

#### 18. Bean的初始化 -> AbstractAutowireCapableBeanFactory.initializeBean()

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			//若我们的bean实现了XXXAware接口进行方法的回调
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			//调用我们的bean的后置处理器的postProcessorsBeforeInitialization方法  @PostCust注解的方法
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			//调用初始化方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			//调用我们bean的后置处理器的PostProcessorsAfterInitialization方法
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}

	private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			//我们的bean实现了BeanNameAware
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			//实现了BeanClassLoaderAware接口
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			//实现了BeanFactoryAware
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}


/**
 * 初始化时调用的 Aware
 * 除了以下三种 Aware 在此处调用，
 * 其他 Aware 将会在 ApplicationContextAwareProcessor.invokeAwareInterfaces() 中调用
 * ApplicationContextAwareProcessor 也是一个 Bean 的后置处理器
 */
private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			//我们的bean实现了BeanNameAware
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			//实现了BeanClassLoaderAware接口
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			//实现了BeanFactoryAware
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```

```java
/**
 * 调用其余 Aware 的函数入口
 * 在这里调用 @PostConstruct 注解标注的方法
 */
@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		//获取我们容器中的所有的bean的后置处理器
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			//挨个调用我们的bean的后置处理器的postProcessBeforeInitialization
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			//若只有有一个返回null 那么直接返回原始的
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

```java
/**
 * 其余 Aware 调用的函数
 */
private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```

```java
/**
 * 对于实现 InitializingBean 的 Bean 的初始化回调
 * 或者对 @Bean(initMehtod="xxx") 等指定 Bean 初始化函数的回调
 */
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		//判断我们的容器中是否实现了InitializingBean接口
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				//回调InitializingBean的afterPropertiesSet()方法
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			//我们beanclass中看是否有自己定义的init方法
			String initMethodName = mbd.getInitMethodName();
			//判断自定义的init方法名称不叫afterPropertiesSet
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				//调用我们自己的初始化方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

#### 19.  Bean 创建完毕，加入缓存池 -> DefaultSingletonBeanRegistry.addSingleton()

```java
/**
	 * 把对象加入到单例缓存池中(所谓的一级缓存 并且考虑循环依赖和正常情况下,移除二三级缓存)
	 * @param beanName bean的名称
	 * @param singletonObject 创建出来的单实例bean
	 */
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			//加入到单例缓存池中
			this.singletonObjects.put(beanName, singletonObject);
			//从三级缓存中移除(针对的不是处理循环依赖的)
			this.singletonFactories.remove(beanName);
			//从二级缓存中移除(循环依赖的时候 早期对象存在于二级缓存)
			this.earlySingletonObjects.remove(beanName);
			//用来记录保存已经处理的bean
			this.registeredSingletons.add(beanName);
		}
	}
```

