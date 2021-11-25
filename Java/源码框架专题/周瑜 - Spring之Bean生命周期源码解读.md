[TOC]

# 周瑜 - Spring之Bean生命周期源码解读

## 导论

- Bean的生命周期就是指：
  - 在Spring中，一个Bean是如何生成的，如何销毁的

## 资料

[JFR介绍](https://zhuanlan.zhihu.com/p/122247741)

[Bean的生命周期流程图](https://www.processon.com/view/link/619c9f4df346fb5facd465eb)

## 流程图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637654383311.png)

## Bean的生成过程

### 1. 生成BeanDefinition

- Spring启动的时候，会进行扫描，调用如下方法：
  - 扫描某个包路径，并得到 BeanDefinition 的 Set 集合

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637654810269.png)

- [扫描流程图](https://www.processon.com/diagraming/619ca650e401fd183dbe35ff)

#### 扫描流程

1. 首先，通过 `ResourcePatternResolver` 获得指定包路径下的所有 `.class` 文件
   - Spring源码中将这些文件包装成了 `Resource` 对象
2. 遍历每个 `Resource` 对象
3. 利用 `MetadataReaderFactory` 解析 `Resource` 对象，得到 `MetadataReader`
   - Spring源码中，MetadataReaderFactory 具体实现类为 `CachingMetadataReaderFactory`
   - MetadataReader 具体实现类为 `SimpleMetadataReader`
4. 利用 `MetadataReader` 进行 `excludeFilters` 和 `includeFilters`，以及条件注解 `@Conditional` 的筛选
   - 某个类上是否存在 @Conditional 注解
     - 存在则调用注解中，所指定的类的 `match` 方法进行匹配
       - 匹配成功则通过筛选
       - 匹配失败则pass掉
5. 筛选通过后，基于 `MetadataReader` 生成 `ScannedGenericBeanDefinition`
6. 再基于 `MetadataReader` 判断，对应的类是不是 接口或抽象类
7. 如果筛选通过，表示扫描到了一个bean，将 `ScannedGenericBeanDefinition` 加入结果集

#### MetadataReader功能

- 表示类的元数据读取器，主要包含一个 `AnnotationMetadata`
- 功能有：
  1. 获取类的名字
  2. 获取父类的名字
  3. 获取所实现的所有接口名
  4. 获取所有内部类的名字
  5. 判断是不是抽象类
  6. 判断是不是接口
  7. 判断是不是一个注解
  8. 获取拥有某个注解的方法集合
  9. 获取类上添加的所有注解信息
  10. 获取类上添加的所有注解类型集合

#### 注意

- `CachingMetadataReaderFactory` 解析某个 .class 文件，得到 MetadataReader 对象
  - 是使用的 `ASM` 技术
  - 并没有加载这个类到 JVM
- 并且，最终得到的 `ScannedGenericBeanDefinition` 对象
  - beanClass 属性，存储的是当前类的名字，而不是 class 对象
  - beanClass 的类型是 Object
- 上面说的是，通过扫描得到 `BeanDefinition`
  - 还可以直接定义 BeanDefinition
  - 或解析 spring.xml 中的 <bean></bean>
  - 或@Bean注解得到的 BeanDefinition

### 2. 合并BeanDefinition

- 通过扫描，得到所有 BeanDefinition 后
  - 就可以根据 BeanDefinition 创建 bean 对象了
- Spring 中支持 父子BeanDefinition
  - 实际使用较少
  - 案例如下：

```xml
<!-- child是单例bean -->
<bean id="parent" class="com.zhouyu.service.Parent" scope="prototype"/>
<bean id="child" class="com.zhouyu.service.Child"/>
```

```xml
<!-- child是原型bean -->
<bean id="parent" class="com.zhouyu.service.Parent" scope="prototype"/>
<bean id="child" class="com.zhouyu.service.Child" parent="parent"/>
```

- child 会继承 parent 上所定义的 scope 属性
- 在根据 child 生成 bean 对象之前
  - 需要进行 BeanDefinition 合并，得到完整的 child 的 BeanDefinition

### 3. 加载类

- BeanDefinition 合并后，就可以创建 bean 对象了
  - 创建 bean 就必须实例化对象
  - 实例化就必须先加载当前 BeanDefinition 所对应的 class
- 在 `AbstractAutowireCapableBeanFactory # createBean()` 中
  - 一开始就会调用：

```java
// 加载类
Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
```

```java
// 方法实现
if (mbd.hasBeanClass()) {
	return mbd.getBeanClass();
}
if (System.getSecurityManager() != null) {
	return AccessController.doPrivileged((PrivilegedExceptionAction<Class<?>>) () ->
		doResolveBeanClass(mbd, typesToMatch), getAccessControlContext());
	}
else {
	return doResolveBeanClass(mbd, typesToMatch);
}
```

```java
public boolean hasBeanClass() {
	return (this.beanClass instanceof Class);
}
```

- 如果 beanClass 属性的类型是 Class，则直接返回
  - 如果不是，则根据类名进行加载
  - doResolveBeanClass() 做的事
- 会利用 BeanFactory 所设置的类加载器来加载类
  - 如果没有设置，则默认使用 `ClassUtils.getDefaultClassLoader()` 所返回的类加载器来加载
- `ClassUtils.getDefaultClassLoader()` :
  - 优先返回当前线程中的 ClassLoader
  - 线程中类加载器为 null 的情况下，返回 ClassUtils 的类加载器
  - 如果 ClassUtils 的类加载器为空，则表示 Bootstrap 加载的
    - 那么返回 系统类加载器

### 4. 实例化前

- 当前 BeanDeifinition 对应的类加载成功后，就可以实例化对象了
- 在Spring中，实例化对象之前，提供了一个扩展点
  - `InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()`
  - 允许用户控制，是否在某个或某些 bean 实例化之前，做一些启动动作
  - 举例：

```java
// 在 userService 实例化前，进行打印
@Component
public class ZhouyuBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
		}
		return null;
	}
}
```

- `postProcessBeforeInstantiation()` 是有返回值的
  - 举例如下，直接返回自定义的一个 UserService 对象
  - 表示不需要 Spring 来实例化了，后续的 依赖注入 也不会进行了
  - 跳过一些步骤，直接执行 初始化后 这一步

```java
@Component
public class ZhouyuBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("实例化前");
			return new UserService();
		}
		return null;
	}
}
```

### 5. 实例化

- 根据 BeanDefinition 创建一个对象

#### 5.1 Supplier创建对象

- 首先判断 BeanDefinition 中是否设置了 `Supplier`

  - 如果设置了则调用 Supplier 的 get() 得到对象
- 得直接使用 BeanDefinition 对象来设置 Supplier
  - 举例：
  

```java
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
beanDefinition.setInstanceSupplier(new Supplier<Object>() {
	@Override
	public Object get() {
		return new UserService();
	}
});
context.registerBeanDefinition("userService", beanDefinition);
```

####   5.2 工厂方法创建对象

- 如果没有设置 Supplier，则检查 BeanDefinition 中是否设置了 factoryMethod，即工厂方法

- 有两种方式可以设置 factoryMethod

  - 方式一：

    - ```xml
      <bean id="userService" class="com.zhouyu.service.UserService" factory-method="createUserService" />
      ```
      
    - ```java
      public class UserService {
      
      	public static UserService createUserService() {
      		System.out.println("执行createUserService()");
      		UserService userService = new UserService();
      		return userService;
      	}
      
      	public void test() {
      		System.out.println("test");
      	}
      
      }
      ```
      

  - 方式二：
  
    - ```xml
      <bean id="commonService" class="com.zhouyu.service.CommonService"/>
      <bean id="userService1" factory-bean="commonService" factory-method="createUserService" />
      ```
  
    - ```java
      public class CommonService {
      
      	public UserService createUserService() {
      		return new UserService();
      	}
      }
      ```

- Spring发现当前 BeanDefinition 设置了 factoryMethod 后，就会区分这两种方式，然后调用 factoryMethod 得到对象
- 注意：
  - 通过 @Bean 定义的 BeanDefinition，是存在 factoryMethod 和 factoryBean 的
  - 和上面的方式二非常类似
    - @Bean 所注解的方法就是 factoryMethod
    - AppConfig 对象就是 factoryBean
  - 如果 @Bean 所注解的方式是 static 的，那么对应的就是 方式一

#### 5.3 推断构造方法

- 推断完构造方法后，就使用构造方法来进行实例化
- 在推断构造方法逻辑中，除了 选择构造方法 以及 查找入参对象外
  - 还会判断 在对应的类中是否存在 `@Lookup` 注解的方法
  - 如果存在，则将该方法封装为 `LookupOverride` 对象、并添加到 BeanDefinition 中
- 在实例化时，如果判断出来当前 BeanDefinition 中没有 LookupOverride
  - 就直接用 构造方法 反射，得到一个实例对象
  - 如果存在 LookupOverride 对象，也就是类中存在 @Lookup 注解的方法，就会生成一个代理对象
- @Lookup 注解就是 `方法注入`，使用 demo 如下：

```java
@Component
public class UserService {

	private OrderService orderService;

	public void test() {
		OrderService orderService = createOrderService();
		System.out.println(orderService);
	}

	@Lookup("orderService")
	public OrderService createOrderService() {
		return null;
	}

}
```

### 6. BeanDefinition的后置处理

- Bean对象实例化出来后，就该给对象的属性进行赋值了
- 在真正属性赋值之前，Spring又提供了一个扩展点：
  - `MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinitio()`
  - 可以对此时的 BeanDefinition 进行加工，举例如下：

```java
@Component
public class ZhouyuMergedBeanDefinitionPostProcessor implements MergedBeanDefinitionPostProcessor {

	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		if ("userService".equals(beanName)) {
			beanDefinition.getPropertyValues().add("orderService", new OrderService());
		}
	}
}
```

- Spring源码中，`AutowiredAnnotationBeanPostProcessor` 就是一个 `MergedBeanDefinitionPostProcessor`
  - 它的 `postProcessMergedBeanDefinition()` 中会去查找注入点，并缓存在 `AutowiredAnnotationBeanPostProcessor` 对象中的一个 Map 中（injectionMetadataCache）

### 7. 实例化后

- 在处理完 BeanDefinition 后，Spring 又设计了一个扩展点：
  - `InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()`
  - 对实例化出来的对象进行处理（这个扩展点，Spring源码中基本没怎么使用）
  - 举例如下：

```java
@Component
public class ZhouyuInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {

		if ("userService".equals(beanName)) {
			UserService userService = (UserService) bean;
			userService.test();
		}

		return true;
	}
}
```

### 8. 自动注入

- 这里指 Spring自动注入

### 9. 处理属性

- 这个步骤中，就会处理 @Autowired、@Resource、@Value 等注解
- 也是通过 `InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()` 实现的
- 案例：实现一个自己的自动注入功能

```java
@Component
public class ZhouyuInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			for (Field field : bean.getClass().getFields()) {
				if (field.isAnnotationPresent(ZhouyuInject.class)) {
					field.setAccessible(true);
					try {
						field.set(bean, "123");
					} catch (IllegalAccessException e) {
						e.printStackTrace();
					}
				}
			}
		}

		return pvs;
	}
}
```

### 10. 执行Aware

- 属性赋值之后，Spring会执行一些回调，包括：
  1. BeanNameAware：回传 beanName 给 bean
  2. BeanClassLoaderAware：回传 classLoader 给 bean
  3. BeanFactoryAware：回传 beanFactory 给对象

### 11. 初始化前

- 也是 Spring 提供的一个扩展点
  - `BeanPostProcessor.postProcessBeforeInitialization()`
  - 举例：

```java
@Component
public class ZhouyuBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化前");
		}

		return bean;
	}
}
```

-  在初始化前，可以对进行了 依赖注入 的 bean 做处理
- 在 Spring 源码中
  1. `InitDestroyAnnotationBeanPostProcessor` 会在初始化前这个步骤，执行 @PostConstruct 方法
  2. `ApplicationContextAwareProcessor` 会在初始化前这个步骤，进行其他 Aware 的回调
     1. EnviromentAware：回传环境变量
     2. EmbeddedValueResolverAware：回传占位符解析器
     3. ResourceLoaderAware：回传资源加载器
     4. ApplicationEventPublisherAware：回传事件发布器
     5. MessageSourceAware：回传国际化资源
     6. ApplicationStartupAware：回传应用其他监听对象，可忽略
     7. ApplicationContextAware：回传Spring容器ApplicationContext

### 12. 初始化

1. 查看当前 bean 对象是否实现了 `InitializingBean` 接口
   1. 如果实现了，就调用其  `afterPropertiesSet()` 方法
2. 执行 BeanDefinition 中指定的初始化方法

### 13. 初始化后

- bean 创建生命周期中的最后一个步骤，Spring也提供了一个扩展点
  - `BeanPostProcessor.postProcessAfterInitialization()`
  - 举例如下：

```java
@Component
public class ZhouyuBeanPostProcessor implements BeanPostProcessor {

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化后");
		}

		return bean;
	}
}
```

- 在这个步骤中，对 bean 最终进行处理
  - Spring 中 AOP 就是基于 `初始化后` 实现的
  - 初始化后返回的对象，才是最终的 bean 对象

## 总结BeanPostProcessor

1. InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()
2. 实例化
3. MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
4. InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
5. 自动注入
6. InstantiationAwareBeanPostProcessor.postProcessProperties()
7. Aware对象
8. BeanPostProcessor.postProcessBeforeInitialization()
9. 初始化
10. BeanPostProcessor.postProcessAfterInitialization()

## Bean的销毁过程

- bean的销毁，是发生在 Spring 容器关闭过程中的
- 举例如下：

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
UserService userService = (UserService) context.getBean("userService");
userService.test();

// 容器关闭
context.close();
```

### DisposableBean

- 实例化之后，会有一个步骤，判断当前创建的 bean 是不是 DisposableBean
  1. 当前 bean 是否实现了 DisposableBean
  2. 或者，当前 bean 是否实现了 AutoCloseable 接口
  3. BeanDefinition 中是否指定了 destroyMethod
  4. 调用 DestructionAwareBeanPostProceesor.requiresDestruction(bean) 进行判断
     1. ApplicationListenerDetector 中，直接使得 ApplicationListener 是 DisposableBean
     2. InitDestroyAnnotationBeanPostProcessor 中，使得 @PreDestroy 注解了的方法，就是 DisposableBean
  5. 把符合上述任意一个条件的 bean，适配成 DisposableBeanAdapter 对象
     1. 并存入 disposableBeans 中（一个 LinkedHashMap）

### Spring容器关闭过程

1. 首先发布 ContextClosedEvent 事件
2. 调用 lifecycleProcessor 的 onCloese() 方法
3. 销毁单例 bean
   1. 遍历 disposableBeans
      1. 将每个遍历到的 disposableBean 从单例池中移除
      2. 调用 disposableBean 的 destroy()
      3. 如果这个 disposableBean 还被其他 bean 依赖了，那么也得销毁其他 bean
      4. 如果这个 disposableBean 还包含了 inner beans，将这些 bean 从单例池中移除掉
         - [inner bean参考资料](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-inner-beans)
   2. 清空 manualSingletonNames（一个Set，存的用户手动注册的单例bean 的 beanName）
   3. 清空 allBeanNamesByType（一个Map，key是bean类型，value是该类型所有的beanName数组）
   4. 清空 singletonBeanNamesByType，和 allBeanNamesByType 类似，只不过只存了单例bean

#### 涉及到适配器模式

- 销毁时，Spring会找出实现了 DisposableBean 接口的 bean
- 但在定义 bean 时，以下几种方式的bean、都需要调用相应销毁方法
  - 实现了 DisposableBean 接口
  - 实现了 AutoCloseable 接口
  - BeanDefinition 中指定了 destroyMethodName
- 所以，这里就需要适配，就用上了 `DisposableBeanAdapter`
  - 将上面的 bean 类都封装成 DisposableBeanAdapter
  - 而 DisposableBeanAdapter 实现了 DisposableBean 接口

