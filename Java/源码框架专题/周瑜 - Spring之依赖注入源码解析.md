[TOC]

# 周瑜 - Spring之依赖注入源码解析

## 资料

[Bean依赖注入流程图](https://www.processon.com/view/link/619ef3b30791295908f4649c)

## 依赖注入的方式

### 手动注入

- xml中定义bean时，就是手动注入

#### set方法注入

```xml
<bean name="userService" class="com.luban.service.UserService">
	<property name="orderService" ref="orderService"/>
</bean>
```

#### 构造注入

```xml
<bean name="userService" class="com.luban.service.UserService">
	<constructor-arg index="0" ref="orderService"/>
</bean>
```

### 自动注入

#### Xml的autowire自动注入

##### 指定自动注入模式

- xml中，在定义bean时，可以指定该bean的自动注入模式：
  - byType
  - byName
  - constructor
  - default
  - no
- byType举例：
  - 表示 Spring 会自动给 userService 中所有属性自动赋值
  - 不需要属性上有 @Autowired 注解，但需要属性有对应的 set 方法
  - 获取到 set 方法唯一参数类型，根据该类型去容器中找 bean
  - 找到多个则报错

```xml
<bean id="userService" class="com.luban.service.UserService" autowire="byType"/>
```

- byName
  - 找到所有 set 方法对应的 xxx 的名字
  - 根据名字去容器中找 bean
- constructor
  - 相当于 byType + byName
  - 根据构造方法中的参数类型找 bean，找到多个则根据参数名确定
- default
  - 表示默认值，<beans> 标签中设置了autowire的值
  - <bean>标签中的 autowire值为default，则表示使用 <beans>标签中的 autowire值
- no
  - 表示关闭 autowire

##### PropertyDescriptor

- 填充属性时，Spring会解析当前类
  - 将当前类所有方法都解析，得到对应的 PropertyDescriptor 对象
- PropertyDescriptor 属性：
  - name：不是方法名，是方法名处理后的名字
    - getXxx 的方法，name = xxx
    - isXxx 的方法，name = xxx
    - setXxx 的方法，name = xxx
  - readMethodRef：表示 get 方法的 Method 对象的引用
  - readMethodName：表示 get 方法的名字
  - writeMethodRef：表示 set 方法的 Method 对象的引用
  - writeMethodName: 表示 set 方法的名字
  - propertyTypeRef：
    - 如果 get 方法，即返回值类型
    - 如果 set 方法，即唯一参数类型
- get方法定义：
  - 方法参数个数为0，并且 get 或 (is 开头 且 返回类型为 boolean)
- set方法定义：
  - 方法参数个数为1，并且 set 开头 且 返回类型为 void

#### @Autowired注解的自动注入

- byType 和 byName 的结合
- 使用场景：
  - 属性上：先根据 属性类型 找 bean，找到多个则根据 属性名 确定一个
  - 构造方法上：先根据方法 参数类型 找 bean，找到多个则根据 参数名 确定一个
  - set方法上：先根据方法 参数类型 找 bean，找到多个则根据 参数名 确定一个

## 寻找注入点

- 创建 bean 的过程中，Spring会利用 `AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition()`  找出注入点并缓存

- 寻找注入点流程为：

  1. 遍历当前类的所有属性字段 Field
     1. 如果 Field 上存在 @Autowired、@Value、@Inject 中的任意一个，则认为该 Filed 是一个注入点
     2. 如果字段是 static 的，则不进行注入
     3. 获取 @Autowired 中 required 的值
     4. 将字段信息构造成一个 `AutowiredFieldElement` 对象，作为一个注入点对象，添加到 `currElements` 集合中
  2. 遍历当前类的所有方法 Method
     1. 判断当前 Method 是否为 `桥接方法`，如果是，则找到原方法
     2. 如果 Method 上存在 @Autowired、@Value、@Inject 中的任意一个，则认为该 Method 是一个注入点
     3. 如果方法是 static 的，则不进行注入
     4. 获取 @Autowired 中的 required 属性的值
     5. 将方法信息构造成一个 `AutowiredMethodElement` 对象，作为一个注入点对象，添加到 `currElements` 集合中
  3. 遍历完当前类的字段和方法后，将遍历父类的，直到没有父类
  4. 最后将 `currElements` 集合封装成一个 InjectionMetadata 对象，作为当前 bean 的注入点集合对象，并缓存

### static注入为什么不行

- 举例：

```java
@Component
@Scope("prototype")
public class OrderService {


}
```

```java
@Component
@Scope("prototype")
public class UserService  {

	@Autowired
	private static OrderService orderService;

	public void test() {
		System.out.println("test123");
	}

}
```

```java
UserService userService1 = context.getBean("userService")
UserService userService2 = context.getBean("userService")
```

- 如上，两个 bean 都是原型bean
- static 是类属性或类方法（属于类），不是实例属性或实例方法
- 那么 userService2 创建成功后，userService1 的 orderService 属性值就变了，这就可能出现bug

### 桥接方法

- 举例如下：

```java
public interface UserInterface<T> {
	void setOrderService(T t);
}
```

```java
@Component
public class UserService implements UserInterface<OrderService> {

	private OrderService orderService;

	@Override
	@Autowired
	public void setOrderService(OrderService orderService) {
		this.orderService = orderService;
	}

	public void test() {
		System.out.println("test123");
	}

}
```

- UserService 对应的字节码：

```java
// class version 52.0 (52)
// access flags 0x21
// signature Ljava/lang/Object;Lcom/zhouyu/service/UserInterface<Lcom/zhouyu/service/OrderService;>;
// declaration: com/zhouyu/service/UserService implements com.zhouyu.service.UserInterface<com.zhouyu.service.OrderService>
public class com/zhouyu/service/UserService implements com/zhouyu/service/UserInterface {

  // compiled from: UserService.java

  @Lorg/springframework/stereotype/Component;()

  // access flags 0x2
  private Lcom/zhouyu/service/OrderService; orderService

  // access flags 0x1
  public <init>()V
   L0
    LINENUMBER 12 L0
    ALOAD 0
    INVOKESPECIAL java/lang/Object.<init> ()V
    RETURN
   L1
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L1 0
    MAXSTACK = 1
    MAXLOCALS = 1

  // access flags 0x1
  public setOrderService(Lcom/zhouyu/service/OrderService;)V
  @Lorg/springframework/beans/factory/annotation/Autowired;()
   L0
    LINENUMBER 19 L0
    ALOAD 0
    ALOAD 1
    PUTFIELD com/zhouyu/service/UserService.orderService : Lcom/zhouyu/service/OrderService;
   L1
    LINENUMBER 20 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L2 0
    LOCALVARIABLE orderService Lcom/zhouyu/service/OrderService; L0 L2 1
    MAXSTACK = 2
    MAXLOCALS = 2

  // access flags 0x1
  public test()V
   L0
    LINENUMBER 23 L0
    GETSTATIC java/lang/System.out : Ljava/io/PrintStream;
    LDC "test123"
    INVOKEVIRTUAL java/io/PrintStream.println (Ljava/lang/String;)V
   L1
    LINENUMBER 24 L1
    RETURN
   L2
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L2 0
    MAXSTACK = 2
    MAXLOCALS = 1

  // access flags 0x1041
  public synthetic bridge setOrderService(Ljava/lang/Object;)V
  @Lorg/springframework/beans/factory/annotation/Autowired;()
   L0
    LINENUMBER 11 L0
    ALOAD 0
    ALOAD 1
    CHECKCAST com/zhouyu/service/OrderService
    INVOKEVIRTUAL com/zhouyu/service/UserService.setOrderService (Lcom/zhouyu/service/OrderService;)V
    RETURN
   L1
    LOCALVARIABLE this Lcom/zhouyu/service/UserService; L0 L1 0
    MAXSTACK = 2
    MAXLOCALS = 2
}

```

- 字节码文件中，存在两个 setOrderService()，并都存在 @Autowired 注解
- Spring遍历到这种桥接方法时，需要找到原方法

## 注入点进行注入

- Spring在 `AutowiredAnnotationBeanPostProcessor.postProcessProperties()` 方法中，会遍历找到的注入点，依次进行注入

### 字段注入

1. 遍历所有 `AutowiredFieldElement` 对象
2. 将对应字段封装为 `DependencyDescriptor` 对象
3. 调用 BeanFactory 的 `resolveDependency()` 方法
   1. 传入 DenpendencyDescriptor 对象
   2. 进行依赖查找，找到当前字段所匹配 bean 对象
4. 将 DependencyDescriptor 对象、和找到的结果对象 beanName
   1. 封装成一个 ShortcutDependencyDescriptor 对象作为缓存
5. 利用反射，将结果对象赋值给字段

### set方法注入

1. 遍历所有 `AutowiredMethodElement` 对象
2. 将对应方法每个参数，封装成 `MethodParameter` 对象
3. 将 `MethodParameter` 对象封装为 `DependencyDescriptor` 对象
4. 调用 BeanFactory 的 `resolveDependency()` 方法
   1. 传入 DenpendencyDescriptor 对象
   2. 进行依赖查找，找到当前字段所匹配 bean 对象
5. 将 DependencyDescriptor 对象、和找到的结果对象 beanName
   1. 封装成一个 ShortcutDependencyDescriptor 对象作为缓存
6. 利用反射，将找到的所有结果对象，入参传给当前方法，并执行

## resolveDependency()

```java
@Nullable
Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
		@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
```

- 传入一个依赖描述（DependencyDescriptor），从 BeanFactory 中找出唯一的一个 bean 对象
- **DefaultListableBeanFactory.resolveDependency()**

[根据type找beanName](https://www.processon.com/view/link/619f3d647d9c083e98ae4a5e)

[根据type找bean](https://www.processon.com/diagraming/619f2b05e401fd48c0ae0389)

### 流程

1. 找出 BeanFactory 中，类型为 type 的所有 beanName
2. 把 `resolvableDependencies()` 中 key 为 type 的对象找出来，并添加到 result 中
3. 遍历1中找到的beanName，判断当前 beanName 对应的 bean 是否能被自动注入
4. 先判断 beanName 对应的 BeanDefinition 中的 autowireCandidate 属性
   1. false：不能被用来进行自动注入
   2. true：则继续判断
5. 判断当前 type 是不是泛型
   1. 是：需要把容器中所有 beanName 拿来遍历
      1. 获取到泛型的真正类型，然后进行匹配
      2. 如果当前 beanName 和当前泛型对应的真实类型匹配，则继续判断
6. 如果当前 DependencyDescriptor 上存在 @Qualifier 注解
   1. 则判断当前 beanName 上是否定义了 Qualifier
   2. 并是否和当前 DependencyDescriptor 上的 Qualifier 相等，相等则匹配
7. 经过上述验证后，当前 beanName 成为一个可注入的，添加到 result 中
