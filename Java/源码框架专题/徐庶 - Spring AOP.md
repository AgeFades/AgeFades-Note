[TOC]

# 徐庶 - Spring AOP

## 画图

[AOP核心概念图](https://www.processon.com/view/link/5ecca5ebe0b34d5f262eae3a)

[AOP切面解析图](https://www.processon.com/view/link/5f1958a35653bb7fd24d0aad)

[AOP创建代理图](https://www.processon.com/view/link/5f1e93f25653bb7fd2549b7c)

[AOP调用代理图](https://www.processon.com/view/link/5f4dd513e0b34d1abc735998)

## 简介

- `AOP` 指在原有代码基础上，做一些增强功能处理，如:
  - 方法执行前
  - 方法返回后
  - 方法抛出异常后
  - ...
- `AOP` 的实现并不是像 Vue 在生命周期里提供钩子方法，
  - 靠的是实现 `代理`，实际运行的实例其实是 `生成的代理类的实例 `

## 核心概念

### 切面

- `Aspect`：指关注点模块化，横切多个对象，如:
  - LogAspect
  - Spring事务管理
  - ...

### 连接点

- `Join point`：代表增强的方法，如:
  - 如 UserController.add() 被 LogAspect 横切，add() 就被称为连接点

## 通知

- `Advice`：在切面某个连接点上执行的增强动作，有多种类型的通知:
  - @Before
  - @After
  - @Around
  - ...

### 目标对象

- `Target`：指要被增强的对象，如:
  - UserController 被 LogAspcet 横切，UserController 就是目标对象

### 切点

- `Pointcut`：匹配连接点的断言
  - 通知 和 切点表达式 相关联，并在满足这个切点的连接点上运行
  - 如:  @Pointcut("@annotation(org.springframework.web.bind.annotation.GetMapping)")

### 顾问

- `Advisor`：是 `Pointcut` 以及 `Advice` 的一个结合包装类，Spring内部封装的类，应用无需关心

### 织入

- `Weaving`：将 `Advice` 切入 `Pointcut` 的过程

### 引入

- `Introductions`：将其他接口和实现，动态引入进 targetClass 中
  - 如: User 有 add()、edit() 方法，引入 SuperUser，即拥有 work() 方法
  - 就是对目标类补充方法的意思，一般没什么用，Spring内部会有使用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618886902033.png)

## 代码案例

```java
@Slf4j
@Aspect
@Component
public class LogAspect {

    private final static int MAX_STR_LENGTH = 500;

    /**
     * 切入点
     */
    @Pointcut("@annotation(org.springframework.web.bind.annotation.GetMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.PostMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.PutMapping) ||" +
            "@annotation(org.springframework.web.bind.annotation.DeleteMapping) ||" +
            "@annotation(com.agefades.single.common.annotation.DoLog)")
    public void log() {
    }

    /**
     * 在所有 Controller 方法作增强方法
     */
    @Around("log()")
    public Object handleControllerMethod(ProceedingJoinPoint pjp) throws Throwable {
        // 调用目标方法，方法计时，单位毫秒
        TimeInterval timer = DateUtil.timer();
        Object object = pjp.proceed();
        doLog(pjp, null, object, timer.interval());
        return object;
    }

    @AfterThrowing(pointcut = "log()", throwing = "e")
    public void logThrowing(JoinPoint joinPoint, Throwable e) {
        doLog((ProceedingJoinPoint) joinPoint, e.getMessage(), null, null);
    }

    private void doLog(ProceedingJoinPoint pjp, String message, Object result, Long time) {
        // 获取类名、方法名
        String targetName = pjp.getSignature().getDeclaringTypeName();
        String className = targetName.substring(targetName.lastIndexOf(".") + 1);
        String methodName = pjp.getSignature().getName();

        log.info("【请求参数】:" + className + ":" + methodName + ": " + AspectUtil.getParams(pjp));
        if (StrUtil.isNotBlank(message)) {
            log.info("【异常信息】:" + className + ":" + methodName + ": " + message);
        } else {
            String resultStr = JSONUtil.toJsonStr(result);
            if (StrUtil.isNotBlank(resultStr) && resultStr.length() > MAX_STR_LENGTH) {
                resultStr = resultStr.substring(0, MAX_STR_LENGTH) + "......";
            }
            log.info("【响应结果】:" + className + ":" + methodName + ": " + resultStr);
            log.info("【方法耗时】:" + className + ":" + methodName + ": 耗时: " + time + "毫秒");
        }
    }

}
```

## Spring AOP 和 AspectJ 异同

### Spring AOP

- 基于 `动态代理` 实现
  - 实现接口时，使用 JDK动态代理实现（默认）
  - 基于继承时，使用 CGLIB 动态代理实现
- 依赖于 IOC 容器，只能作用于 Spring IOC 容器中的 Bean
- 容器启动时，需要生成代理实例，在方法调用上，会增加 栈的深度，性能不如 AspectJ
- Spring AOP 已经基本满足普通应用开发 对切面 的使用需求

### AspectJ

- 属于 `静态织入`，通过修改代码实现，织入时机有:
  - `Compile-time weaving`：编译期织入
  - `Post-compile weaving`：编译后织入
  - `Load-time weaving`：类加载织入
- 功能比 Spring AOP 更加强大，性能更好，但使用更加复杂（所以我选Spring AOP）

## Spring AOP使用

- 目前 Spring AOP 有三种配置方式

  - Spring1.2 `基于接口配置`
    - 课程有代码案例展示，但考虑到现在基本不用，了解即可，故不作记录
  - Spring2.0 `schema-based配置` 
    - 即 XML 配置，<aop></aop>
  - Spring2.0 `基于注解配置`
    - 即现在通用的 @Aspect、@Before...

- Spring AOP 使用了 `责任链模式` 来对 Advice 和 Interceptor 进行调用
- `Advisor`：对 `Advice` 和 `Interceptor` 的包装
  - Advisor 决定拦截哪些方法
  - Advice 对拦截方法进行增强处理
- Advisor 的类型:
  - `NameMatchMethodPointcutAdvisor`：根据名称匹配
  - `RegexpMethodPointcutAdvisor`：根据正则匹配
- 上述横切作用范围太受限，每次还是对单个 Bean 做代理，这里引入 AutoPorxy
  - `BeanNameAutoProxyCreator`：根据Bean名称创建代理
  - `DefaultAdvisorAutoProxyCreator`：自动匹配创建代理,一般搭配 `RegexMethodPointcutAdvisor` 配置

## Spring AOP源码

### 使用现象

1. 切面类上加 @Aspect 注解
2. 定义一个 Pointcut 方法
3. 定义系列增强方法

### 分析思路

1. 找到所有的切面类
2. 解析出所有的 Advice 并保存
3. 创建一个动态代理类
4. 调用目标类方法时，找到其代理类，并依次调用 Advice 对目标方法进行增强

![](https://note.youdao.com/yws/public/resource/30678c0adbb76ce345399b05ea785959/xmlnote/C38E8FA12B1B4B2886053D70201B491E/6492)

## SpringBoot AOP入口

1. 通过 `@EnableAspectJAutoProxy` 开启 AOP 切面
   1. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618889792637.png)
2. `@Import` 引入 `AspectJAutoProxyRegistrar`
   1. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1618890010411.png)
3. `AspectJAutoProxyRegistrar` 实现了 `ImportBeanDefinitionRegistrar`
   1. 可以通过 `registerBeanDefinitions()` 给 IOC 容器注入 BeanDefinition
   2. 类似的操作都是通过 Spring 提供的各种拓展点完成的（Aware、BeanPostProcessor...） 
   3. rergisterAspectJAnnotation...() 这个方法最终注入类为 `AnnotationAwareAspectJAutoProxyCreator`
   4. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619318946381.png)
4. 注入类解析切面
   1. 通过实现 `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()` 
5. 注入类创建代理
   1. 通过实现 `BeanPostProcessor.postProcessAfterInitialization()`
6. 循环依赖时, AOP的处理
   1. 通过实现 `SmartInstantiationAwareBeanPostProcessor.getEarlyBeanReference()`

## 1. 切面类的解析

### postProcessBeforerInstantiation()

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619320966747.png)

```java
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
  // 构建缓存Key
  Object cacheKey = getCacheKey(beanClass, beanName);

  if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
    // 在缓存中存在，表示已经解析过，直接返回null
    if (this.advisedBeans.containsKey(cacheKey)) {
      return null;
    }
    // 判断是否为切面类、通知、切点...
    // 判断是否应该跳过, 默认返回 false，应该被跳过的（老版本通过实现 Advisor 接口的 Bean）
    if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
      // 放入缓存
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      // 如果是切面类、通知、切点，或者应该跳过解析的Bean，这里也是直接返回null
      return null;
    }
  }

  
  TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
  if (targetSource != null) {
    if (StringUtils.hasLength(beanName)) {
      this.targetSourcedBeans.add(beanName);
    }
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
    Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
  }

  return null;
}
```

### shouldSkip()

```java
public class AspectJAwareAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator { 

	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    		// 找出XML配置的Advisor、实现原生接口的Advisor、例: 实现事务相关的Advisor
        List<Advisor> candidateAdvisors = this.findCandidateAdvisors();
    
        for (Advisor advisor : candidateAdvisors) {
        if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
          return true;
        }
      }
      return super.shouldSkip(beanClass, beanName);
    }

}
```

### findCandidateAdvisors()

```java
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {
  
	protected List<Advisor> findCandidateAdvisors() {
		// 找出XML配置的Advisor、实现原生接口的Advisor、例: 实现事务相关的Advisor
		List<Advisor> advisors = super.findCandidateAdvisors();
    
		// 找出Aspect相关信息，并将其封装为一个 Advisor
		if (this.aspectJAdvisorsBuilder != null) {
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
    
    // 返回所有 Advisor
		return advisors;
	}
  
}
```

### buildAspectJAdvisors()

```java
public class BeanFactoryAspectJAdvisorsBuilder { 
	
  /**
   * 拿到所有Bean定义，判断是否标记 @Aspect
   * 将每个通知最终都包装成一个 Advisor
   */
  public List<Advisor> buildAspectJAdvisors() {
    // 用于保存切面名称的缓存，类级别变量缓存
		List<String> aspectNames = this.aspectBeanNames;

		if (aspectNames == null) {
			synchronized (this) {
				aspectNames = this.aspectBeanNames;
				if (aspectNames == null) {
					List<Advisor> advisors = new ArrayList<>();
					aspectNames = new ArrayList<>();
					String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
							this.beanFactory, Object.class, true, false);
          // 遍历IOC容器中所有Bean名称
					for (String beanName : beanNames) {
						if (!isEligibleBean(beanName)) {
							continue;
						}
						
            // 通过 beanName 获取Class对象
						Class<?> beanType = this.beanFactory.getType(beanName);
						if (beanType == null) {
							continue;
						}
            // 根据Class对象判断是否为切面, 方法内部就是判断 Class 对象上有无 @Aspect 注解
						if (this.advisorFactory.isAspect(beanType)) {
              // 是切面、则加入缓存
							aspectNames.add(beanName);
              // 将 beanName 和 Class对象 构建为一个 AspectMetadata
							AspectMetadata amd = new AspectMetadata(beanType, beanName);
							if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                
                // 构建切面注解的实例工厂
								MetadataAwareAspectInstanceFactory factory =
										new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                
                // 获取切面中的通知对象、并包装成 Advisor 如 @Before、@After...
								List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                
                // 加入缓存中
								if (this.beanFactory.isSingleton(beanName)) {
									this.advisorsCache.put(beanName, classAdvisors);
								}
								else {
									this.aspectFactoryCache.put(beanName, factory);
								}
								advisors.addAll(classAdvisors);
							}
							else {
								// Per target or per this.
								if (this.beanFactory.isSingleton(beanName)) {
									throw new IllegalArgumentException("Bean with name '" + beanName +
											"' is a singleton, but aspect instantiation model is not singleton");
								}
								MetadataAwareAspectInstanceFactory factory =
										new PrototypeAspectInstanceFactory(this.beanFactory, beanName);
								this.aspectFactoryCache.put(beanName, factory);
								advisors.addAll(this.advisorFactory.getAdvisors(factory));
							}
						}
					}
					this.aspectBeanNames = aspectNames;
					return advisors;
				}
			}
		}

		if (aspectNames.isEmpty()) {
			return Collections.emptyList();
		}
		List<Advisor> advisors = new ArrayList<>();
		for (String aspectName : aspectNames) {
			List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
			if (cachedAdvisors != null) {
				advisors.addAll(cachedAdvisors);
			}
			else {
				MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
				advisors.addAll(this.advisorFactory.getAdvisors(factory));
			}
		}
		return advisors;
	}
  
}
```

### getAdvisors()

```java
public class ReflectiveAspectJAdvisorFactory extends AbstractAspectJAdvisorFactory implements Serializable {
  
  public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    // 获取标记为 Aspect 的类
		Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    
    // 获取切面类名称
		String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    
    // 校验切面类
		validate(aspectClass);

		// We need to wrap the MetadataAwareAspectInstanceFactory with a decorator
		// so that it will only instantiate once.
		MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
				new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);

		List<Advisor> advisors = new ArrayList<>();
    
    // 获取切面类中所有方法、不会解析 @PointCut 注解标记的方法
		for (Method method : getAdvisorMethods(aspectClass)) {
      
      // 解析通知、并包装成 Advisor
			Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, advisors.size(), aspectName);
      
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		// If it's a per target aspect, emit the dummy instantiating aspect.
		if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
			Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(lazySingletonAspectInstanceFactory);
			advisors.add(0, instantiationAdvisor);
		}

		// Find introduction fields.
		for (Field field : aspectClass.getDeclaredFields()) {
			Advisor advisor = getDeclareParentsAdvisor(field);
			if (advisor != null) {
				advisors.add(advisor);
			}
		}

		return advisors;
	}
  
}
```

- 至此，Spring就完成了对AOP切面的解析操作

### 切面通知的顺序

- 按源码注解顺序
- 责任链调用模式 Advisor

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1619333899347.png)

## 2. 创建动态代理

![](https://note.youdao.com/yws/public/resource/30678c0adbb76ce345399b05ea785959/xmlnote/987699CA0518425095763AE3A6DD52CD/6519)

### postProcessAfterInitialization()

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
  
  @Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
      // 获取缓存key
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
      
      // 如果之前循环依赖创建的动态代理是现在的Bean，则不再创建 并且移除
			if (!this.earlyProxyReferences.contains(cacheKey)) {
        
        // 返回动态代理实例对象
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
  
}
```

### wrapIfNecessary()

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
  	// 已被处理过（解析切面时,targetSourceBeans 出现过），即为自己实现创建动态代理逻辑
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
  
  	// advisedBean 不需要增强
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
  
  	// 是不是基础bean、是不是需要跳过bean
  	// 这里是重复判断，因为在创建Bean时，循环依赖是可以改变Bean的
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// 根据当前Bean找到匹配的 Advisor
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
  
  	// 如果当前Bean匹配到了Advisor
		if (specificInterceptors != DO_NOT_PROXY) {
      // 标记为已处理
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
      
      // 创建真正的代理对象
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      
      // 将代理对象放入缓存
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

## 3. 调用

