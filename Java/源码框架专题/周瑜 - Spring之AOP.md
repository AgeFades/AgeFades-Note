[TOC]

# 周瑜 - Spring之AOP

## 动态代理

### 代理模式概念

- 为 `目标对象` 提供一种 `代理` 以控制对该 `目标对象` 的访问
  - 增强类中的某些方法，对程序进行扩展

### Cglib代理代码案例

- 目标对象

```java
public class UserService  {

	public void test() {
		System.out.println("test...");
	}

}
```

- 创建代理

```java
UserService target = new UserService();

// 通过cglib技术
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserService.class);

// 定义额外逻辑，也就是代理逻辑
enhancer.setCallbacks(new Callback[]{new MethodInterceptor() {
	@Override
	public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
		System.out.println("before...");
		Object result = methodProxy.invoke(target, objects);
		System.out.println("after...");
		return result;
	}
}});

// 动态代理所创建出来的UserService对象
UserService userService = (UserService) enhancer.create();

// 执行这个userService的test方法时，就会额外会执行一些其他逻辑
userService.test();
```

#### Cglib代理方式

- 代理对象的创建，是基于 `父子类 - 即继承` 的
  - 父类：目标类
  - 子类：代理类

### JDK代理代码案例

- 目标类

```java
public interface UserInterface {
	public void test();
}

public class UserService implements UserInterface {

	public void test() {
		System.out.println("test...");
	}

}
```

- 代理类

```java
UserService target = new UserService();

// UserInterface接口的代理对象
Object proxy = Proxy.newProxyInstance(UserService.class.getClassLoader(), new Class[]{UserInterface.class}, new InvocationHandler() {
	@Override
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("before...");
		Object result = method.invoke(target, args);
		System.out.println("after...");
		return result;
	}
});

UserInterface userService = (UserInterface) proxy;
userService.test();
```

#### JDK代理方式

- 代理对象的创建，是基于 `实现接口` 
  - 目标对象必须先实现某些接口
  - 产生的代理对象类型，实际是目标对象实现接口的类型，而不能转为目标对象类型

## ProxyFactory

### 简介

- Spring对以上两种代理技术做了封装，封装的实现类就是 `ProxyFactory`
- 使用 `ProxyFactory`，开发不需要关注代理方式到底是 JDK、还是 Cglib
- ProxyFactory的判断逻辑：
  - 目标对象实现了接口，则使用 JDK 动态代理
  - 目标对象未实现接口，则使用 Cglib 动态代理

### 代码案例

```java
UserService target = new UserService();

ProxyFactory proxyFactory = new ProxyFactory();
proxyFactory.setTarget(target);
proxyFactory.addAdvice(new MethodInterceptor() {
	@Override
	public Object invoke(MethodInvocation invocation) throws Throwable {
		System.out.println("before...");
		Object result = invocation.proceed();
		System.out.println("after...");
		return result;
	}
});

UserInterface userService = (UserInterface) proxyFactory.getProxy();
userService.test();
```

## Advice分类

1. Before Advice：方法之前执行
2. After returning advice：方法return后执行
3. After throwing advice：方法抛异常后执行
4. After finally advice：方法执行完 finally 后执行，比 return 更后
5. Around advice：环绕通知，可自定义执行顺序

## Advisor的理解

- 一个 `Advisor` 是由一个 `Pointcut` 和一个 `Advice` 组成
- 通过 `Pointcut` 指定需要被代理的方法

### Advisor通过Pointcut指定代理方法

```java
		UserService target = new UserService();

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.setTarget(target);
		proxyFactory.addAdvisor(new PointcutAdvisor() {
			@Override
			public Pointcut getPointcut() {
				return new StaticMethodMatcherPointcut() {
					@Override
					public boolean matches(Method method, Class<?> targetClass) {
						return method.getName().equals("testAbc");
					}
				};
			}

			@Override
			public Advice getAdvice() {
				return new MethodInterceptor() {
					@Override
					public Object invoke(MethodInvocation invocation) throws Throwable {
						System.out.println("before...");
						Object result = invocation.proceed();
						System.out.println("after...");
						return result;
					}
				};
			}

			@Override
			public boolean isPerInstance() {
				return false;
			}
		});

		UserInterface userService = (UserInterface) proxyFactory.getProxy();
		userService.test();
```

## Spring中创建代理对象的方式

### ProxyFactoryBean

#### 代码案例

```java
@Bean
public ProxyFactoryBean userServiceProxy(){
	UserService userService = new UserService();

	ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
	proxyFactoryBean.setTarget(userService);
	proxyFactoryBean.addAdvice(new MethodInterceptor() {
		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			System.out.println("before...");
			Object result = invocation.proceed();
			System.out.println("after...");
			return result;
		}
	});
	return proxyFactoryBean;
}

```

- 这种方式能定义一个经过了 AOP 的 bean，但只能针对某一个bean
- 实际得到一个 FactoryBean，间接将 UserService 的代理对象作为了 bean

#### 扩展

- 可以将某个 Advice 或 Advisor 定义为 bean，然后在 ProxyFactoryBean 中进行设置

```java
@Bean
public MethodInterceptor zhouyuAroundAdvice(){
	return new MethodInterceptor() {
		@Override
		public Object invoke(MethodInvocation invocation) throws Throwable {
			System.out.println("before...");
			Object result = invocation.proceed();
			System.out.println("after...");
			return result;
		}
	};
}

@Bean
public ProxyFactoryBean userService(){
	UserService userService = new UserService();
	
    ProxyFactoryBean proxyFactoryBean = new ProxyFactoryBean();
	proxyFactoryBean.setTarget(userService);
	proxyFactoryBean.setInterceptorNames("zhouyuAroundAdvice");
	return proxyFactoryBean;
}
```

### BeanNameAutoProxyCreator

- 通过指定某个 bean 的名字，来对该 bean 进行代理
- 可以对批量 bean 进行 AOP，并指定代理逻辑
  - 指定一个 InterceptorName（即一个 Advice）
  - 前提：该 Advice 必须也是一个 bean，否则找不到

#### 代码案例

```java
@Bean
public BeanNameAutoProxyCreator beanNameAutoProxyCreator() {
	BeanNameAutoProxyCreator beanNameAutoProxyCreator = new BeanNameAutoProxyCreator();
	beanNameAutoProxyCreator.setBeanNames("userSe*");
	beanNameAutoProxyCreator.setInterceptorNames("zhouyuAroundAdvice");
	beanNameAutoProxyCreator.setProxyTargetClass(true);

    return beanNameAutoProxyCreator;
}
```

### DefaultAdvisorAutoProxyCreator

#### 代码案例

```java
@Bean
public DefaultPointcutAdvisor defaultPointcutAdvisor(){
	NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut();
	pointcut.addMethodName("test");

    DefaultPointcutAdvisor defaultPointcutAdvisor = new DefaultPointcutAdvisor();
	defaultPointcutAdvisor.setPointcut(pointcut);
	defaultPointcutAdvisor.setAdvice(new ZhouyuAfterReturningAdvice());

    return defaultPointcutAdvisor;
}

@Bean
public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
	
    DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator = new DefaultAdvisorAutoProxyCreator();

	return defaultAdvisorAutoProxyCreator;
}
```

- 直接找到所有 Advisor 类型的 bean，根据其 PointCut 和 Advice 信息，确定要代理的bean以及代理逻辑

### 通过注解

#### 代码案例

```java
@Aspect
@Component
public class ZhouyuAspect {

	@Before("execution(public void com.zhouyu.service.UserService.test())")
	public void zhouyuBefore(JoinPoint joinPoint) {
		System.out.println("zhouyuBefore");
	}

}
```

- Spring解析这些注解，得到对应的 PointCut、Advice、Advisor 对象，放进 ProxyFactory 中
  - 从而产生对应的代理对象
- 解析注解的逻辑在 `@EnableAspectJAutoProxy` 中

## Spring中Advice注解对应的类

1. @Before：AspectJMethodBeforeAdvice
2. @After：AspectJAfterAdvice
3. @Arond：AspectJAroundAdvice
4. 依次类推...

## Introduction

https://www.cnblogs.com/powerwu/articles/5170861.html

## LoadTimeWeaver

https://www.cnblogs.com/davidwang456/p/5633609.html

## TargetSource的使用

- Spring AOP 提供的一种机制，可以让开发用自定义逻辑创建 `被代理对象`

### 代码案例

```java
protected Object buildLazyResolutionProxy(final DependencyDescriptor descriptor, final @Nullable String beanName) {
		BeanFactory beanFactory = getBeanFactory();
		Assert.state(beanFactory instanceof DefaultListableBeanFactory,
				"BeanFactory needs to be a DefaultListableBeanFactory");
		final DefaultListableBeanFactory dlbf = (DefaultListableBeanFactory) beanFactory;

		TargetSource ts = new TargetSource() {
			@Override
			public Class<?> getTargetClass() {
				return descriptor.getDependencyType();
			}
			@Override
			public boolean isStatic() {
				return false;
			}
			@Override
			public Object getTarget() {
				Set<String> autowiredBeanNames = (beanName != null ? new LinkedHashSet<>(1) : null);
				Object target = dlbf.doResolveDependency(descriptor, beanName, autowiredBeanNames, null);
				if (target == null) {
					Class<?> type = getTargetClass();
					if (Map.class == type) {
						return Collections.emptyMap();
					}
					else if (List.class == type) {
						return Collections.emptyList();
					}
					else if (Set.class == type || Collection.class == type) {
						return Collections.emptySet();
					}
					throw new NoSuchBeanDefinitionException(descriptor.getResolvableType(),
							"Optional dependency not present for lazy injection point");
				}
				if (autowiredBeanNames != null) {
					for (String autowiredBeanName : autowiredBeanNames) {
						if (dlbf.containsBean(autowiredBeanName)) {
							dlbf.registerDependentBean(autowiredBeanName, beanName);
						}
					}
				}
				return target;
			}
			@Override
			public void releaseTarget(Object target) {
			}
		};

		ProxyFactory pf = new ProxyFactory();
		pf.setTargetSource(ts);
		Class<?> dependencyType = descriptor.getDependencyType();
		if (dependencyType.isInterface()) {
			pf.addInterface(dependencyType);
		}
		return pf.getProxy(dlbf.getBeanClassLoader());
	}
```

- 以上代码目的：代理对象在执行某个方法时，调用 TargetSource.getTarget() 实时得到一个 `被代理对象`

## AbstractAdvisorAutoProxyCreator

- 该对象很强大 & 重要
  - 在 Spring 容器中，存在该类型的 bean，就相当于开启了 AOP

- 是 `DefaultAdvisorAutoProxyCreator` 的父类
- 实际上是一个 `BeanPostProcessor`，在创建 bean 时，就会进入到它对应的生命周期方法中
  - 例如：在某个 bean `初始化之后`，会调用 `wrapIfNecessary()` 进行 AOP
  - 底层逻辑：
    1. `AbstractAdvisorAutoProxyCreator` 会找到所有的 Advisor，
    2. 然后根据 PointCut 判断该 bean 是否和某个 Advisor 匹配
    3. 匹配则说明需要进行AOP，从而产生一个代理对象

## @EnableAspectJAutoProxy

- 注解作用：往 Spring 容器中，添加一个 `AnnotationAwareAspectJAutoProxyCreator` 类型的 bean

![](https://cdn.nlark.com/yuque/0/2021/png/365147/1628753226288-643a0eb5-c2a9-4d8a-9d22-a5a47a6ffb0c.png)

- `AnnotationAwareAspectJAutoProxyCreator` 继承 `AbstractAdvisorAutoProxyCreator`
  - 重写了 `findCandidateAdvisors()`
  - 除了本来找到所有 Advisor 类型的 bean 功能以外，还能解析 @Aspect、@Before、@PointCut...注解
  - 最终生成 Advisor 对象

## Spring AOP原理流程图

https://www.processon.com/view/link/61b717e01efad42237ba97f4

