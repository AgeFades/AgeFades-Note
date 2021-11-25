[TOC]

# 周瑜 - Spring之底层架构核心概念解析

## BeanDefinition

### 类属性图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637561158053.png)

### 属性简介

- BeanDefinition 表示 bean 定义，用很多属性来描述一个 bean 的特点，比如：
  - class：表示bean类型
  - scope：表示bean作用域，单例或原型等
  - lazyInit：表示bean是否懒加载
  - initMethodName：表示bean初始化时要执行的方法
  - destroyMethodName：表示bean销毁时要执行的方法
  - ......
- bean最终都被解析成 BeanDefinition、并注册进Spring容器中

### 声明式定义bean

- <bean></bean>
- @Bean
- @Component、@Service、@Controller

### 编程式定义bean

- 直接通过 BeanDefinition，比如：

```java
import org.springframework.beans.factory.support.AbstractBeanDefinition;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;

public class Test {

	public static void main(String[] args) {
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
		registryBeanDefinition(context);
		User user = (User) context.getBean("user");
		user.test();
	}

	public static void registryBeanDefinition(AnnotationConfigApplicationContext context) {
		// 手动新建一个BeanDefinition
		AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
    
    // 设置bean类型
		beanDefinition.setBeanClass(User.class);
    
    // 设置作用域
    beanDefinition.setScope("prototype");
    
    // 设置初始化方法
    beanDefinition.setInitMethodName("init");
    
    // 设置懒加载
    beanDefinition.setLazyInit(true);

		// 注册进IOC
		context.registerBeanDefinition("user", beanDefinition);
	}

	public static class User {
		public void test() {
			System.out.println("编程式定义bean");
		}
 	}

}
```

## BeanDefinitionReader

### 类属性图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637562247295.png)

### 简介

- BeanDefinition读取器
- 业务使用少、Spring源码使用多，相当于Spring源码的基础设施

#### AnnotatedBeanDefinitionReader

- 可以直接把某个类转换为 BeanDefinition
- 并会解析该类上的注解
  - @Conditional
  - @Scope
  - @Lazy
  - @Primary
  - @DependsOn
  - @Role
  - @Description

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(context);

// 将User.class解析为BeanDefinition
annotatedBeanDefinitionReader.register(User.class);

System.out.println(context.getBean("user"));
```

#### XmlBeanDefinitionReader

- 可以解析 <bean></bean> 标签

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(context);
int i = xmlBeanDefinitionReader.loadBeanDefinitions("spring.xml");

System.out.println(context.getBean("user"));
```

#### ClassPathBeanDefinitionScanner

- 扫描器，作用类似 BeanDefinitionReader
- 比如：扫描到类上有 @Component 注解，会将其解析为一个 BeanDefinition

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.refresh();

ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(context);
scanner.scan("com.zhouyu");

System.out.println(context.getBean("userService"));
```

## BeanFactory

### 类属性图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637563884358.png)

### 简介

- 创建bean、提供获取bean的API

### ApplicationContext和BeanFactory的关系

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637563850372.png)

- ApplicationContext 是 BeanFactory 的一个实现类
- 除了 创建bean、提供获取bean的API 功能外，ApplicationContxt 还实现了很多其他接口，有更多功能
  - MessageSource：国际化
  - ApplicationEventPublisher：事件发布
  - EnvironmentCapable：获取环境变量
  - ResourcePatternResolver：资源加载器
  - ListableBeanFactory：获取 beanNames
  - HierarchicalBeanFactory：获取父 BeanFactory
  - ......
- 在 Spring 源码实现中，new ApplicationContext() 底层会 new BeanFacotry()
  - context.getBean() 底层调用的就是 BeanFactory.getBean()

### DefaultListableBeanFactory

- BeanFactory 核心实现类

#### 类继承结构

![](https://cdn.nlark.com/yuque/0/2020/png/365147/1602053031607-fd00a145-67fa-4231-8cca-9186db5f2b00.png)

#### 部分替代ApplicationContext功能

```java
DefaultListableBeanFactory beanFactory = new DefaultListableBeanFactory();

AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
beanDefinition.setBeanClass(User.class);

beanFactory.registerBeanDefinition("user", beanDefinition);

System.out.println(beanFactory.getBean("user"));
```

#### 实现接口功能

##### 1. AliasRegistry

- 支持别名功能，一个名字可以对应多个别名

##### 2. BeanDefinitionRegistry

- 可以 `注册、保存、移除、获取` 某个 BeanDefinition

##### 3. BeanFactory

- 可以根据 bean 名字、或类型、或别名，获取 bean 对象

##### 4. SingletonBeanRegistry

- 可以直接注册、获取某个`单例bean`

##### 5. SimpleAliasRegistry

- 一个类，实现了 AliasRegistry 接口所定义的功能，支持别名功能

##### 6. ListableBeanFactory

- 在 BeanFactory 的基础上，增加了其他功能
  - 可以获取所有 BeanDefinition 的 beanNames
  - 可以根据某个类型获取对应的 beanNames
  - 可以根据某个类型获取 {类型：对应bean} 的映射关系

##### 7. HierarchicalBeanFactory

- 在 BeanFactory 的基础上，添加了获取父 BeanFactory 的功能

##### 8. DefaultSingletonBeanRegistry

- 一个类，实现了 SingletonBeanRegistry 接口
  - 拥有直接注册、获取某个 `单例bean` 的功能

##### 9. ConfigurableBeanFactory

- 在 `HierarchicalBeanFactory` 和 `SingletonBeanRegistry` 的基础上，添加了功能：
  - 设置父BeanFactory
  - 类加载器（表示可以指定某个类加载器，进行类的加载）
  - 设置 SpEL 表达式解析器（表示该 BeanFactory 可以解析 EL表达式）
  - 设置类型转化服务（表示该 BeanFactory 可以进行 类型转化）
  - 可以添加 BeanPostProcessor（表示该 BeanFactory 支持 bean后置处理器）
  - 可以合并 BeanDefinition
  - 可以销毁某个 bean
  - ......

##### 10. FactoryBeanRegistrySupport

- 支持了 FactoryBean 的功能

##### 11. AutowireCapableBeanFactory

- 直接继承 BeanFactory，在其基础上，支持在创建 bean 的过程中，能对 bean 进行自动装配

##### 12. AbstractBeanFactory

- 实现了 `ConfigurableBeanFactory` 接口，继承了 `FactoryBeanRegistrySupport` 
  - 这个 BeanFactory 类功能已经很全面了
  - 但是不能 自动装配 和 获取beanNames

##### 13. ConfigurableListableBeanFactory

- 继承了 `ListableBeanFactory`、`AutowireCapableBeanFactory`、`ConfigurableBeanFactory`

##### 14. AbstarctAutowireCapableBeanFactory

- 继承了 `AbstractBeanFactory`，实现了 `AutowireCapableBeanFactory`，拥有了自动装配功能

##### 15. DefaultListableBeanFactory

- 继承了 `AbstractAutowireCapableBeanFactory`
  - 实现了 `ConfigurableListableBeanFactory` 和 `BeanDefinitionRegistry` 
- 是功能最全面的 BeanFactory

## ApplicationContext

- 简介上面介绍过了，记下两个重要实现类

### AnnotationConfigApplicationContext

![](https://cdn.nlark.com/yuque/0/2020/png/365147/1602055860352-0925b046-b88e-4085-b872-b1ec5aeb8fee.png)

#### 1. ConfigurableApplicationContext

- 继承了 ApplicationContext 接口，增加了：
  - 添加事件监听器
  - 添加 BeanFactoryPostProcessor
  - 设置 Environment
  - 获取 ConfigurableListableBeanFactory
  - ...

#### 2. AbstractApplicationContext

- 实现了 ConfigurableApplication 接口

#### 3. GenericApplicationContext

- 继承了 AbstractApplicationContext，实现了 BeanDefinitionRegistry
- 拥有所有 ApplicationContext 的功能
  - 并且可以注册BeanDefinition
- 注意该类中有一个属性：`DefaultListableBeanFactory`

#### 4. AnnotationConfigRegistry

- 可以单独注册某个类为 BeanDefinition
  - 可以处理 @Configuration 注解
  - 可以处理 @Bean 注解
  - 可以扫描

#### 5. AnnotationConfigApplicationContext

- 继承 GenericApplicationContext、实现 AnnotationConfigRegistry
- 拥有以上所有功能

### ClassPathXmlApplicationContext

![](https://cdn.nlark.com/yuque/0/2020/png/365147/1602056629659-0bb9b513-834c-4e57-9120-55dc40fd8674.png)

- 继承 AbstractApplicationContext
- 相对 AnnotationConfigApplicationContext，功能没有那么全面强大
  - 比如：不能注册 BeanDefinition

### 国际化

- 先定义一个 MessageSource

```java
@Bean
public MessageSource messageSource() {
	ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
	messageSource.setBasename("messages");
	return messageSource;
}
```

- 有了该bean，可以在任意想进行国际化的地方，使用该MessageSource
- ApplicationContext 也拥有国际化的功能，也可以直接这么用：

```java
context.getMessage("test", null, new Locale("en_CN"))
```

### 资源加载

- ApplicationContext 还可以加载资源
- 案例1

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

Resource resource = context.getResource("/Users/agefades/Documents/Project/spring-framework-5.3.10/tuling/src/main/java/com/zhouyu/Test.java");
System.out.println(resource.contentLength());
System.out.println(resource.getFilename());
```

- 案例2

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

Resource resource1 = context.getResource("https://www.baidu.com");
System.out.println(resource1.contentLength());
System.out.println(resource1.getURL());

Resource resource2 = context.getResource("classpath:spring.xml");
System.out.println(resource2.contentLength());
System.out.println(resource2.getURL());
```

- 案例3，一次加载多个资源

```java
Resource[] resources = context.getResources("classpath:com/zhouyu/*.class");
for (Resource resource : resources) {
	System.out.println(resource.contentLength());
	System.out.println(resource.getFilename());
}
```

### 获取运行时环境

```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

Map<String, Object> systemEnvironment = context.getEnvironment().getSystemEnvironment();
System.out.println(systemEnvironment);

System.out.println("=======");

Map<String, Object> systemProperties = context.getEnvironment().getSystemProperties();
System.out.println(systemProperties);

System.out.println("=======");

MutablePropertySources propertySources = context.getEnvironment().getPropertySources();
System.out.println(propertySources);

System.out.println("=======");

System.out.println(context.getEnvironment().getProperty("NO_PROXY"));
System.out.println(context.getEnvironment().getProperty("sun.jnu.encoding"));
System.out.println(context.getEnvironment().getProperty("zhouyu"));
```

#### @PropertySource

- 可以使用该注解，使某个文件中的参数、添加到运行时环境中
- 案例如下：

```java
@PropertySource("classpath:spring.properties")
```

### 事件发布

- 先定义一个事件监听器

```java
@Bean
public ApplicationListener listener() {
  return event -> System.out.println(event.getSource());
}
```

- 然后发布一个事件

```java
context.publishEvent("kkk");
```

## 类型转化

- Spring源码中，可能需要把 String 转成其他类型，所以在 Spring 源码中提供了一些技术来更方便的做 对象的类型转换
- Spring源码中，类型转换的应用场景很多

### PropertyEditor

- JDK提供的类型转化工具类

```java
public class StringToUserPropertyEditor extends PropertyEditorSupport implements PropertyEditor {

	@Override
	public void setAsText(String text) throws IllegalArgumentException {
		User user = new User();
		user.setName(text);
		this.setValue(user);
	}
}
```

```java
StringToUserPropertyEditor propertyEditor = new StringToUserPropertyEditor();
propertyEditor.setAsText("1");
User value = (User) propertyEditor.getValue();
System.out.println(value);
```

#### 向Spring注册PropertyEditor

```java
@Bean
public CustomEditorConfigurer customEditorConfigurer() {
	CustomEditorConfigurer customEditorConfigurer = new CustomEditorConfigurer();
	Map<Class<?>, Class<? extends PropertyEditor>> propertyEditorMap = new HashMap<>();
    
    // 表示StringToUserPropertyEditor可以将String转化成User类型，在Spring源码中，如果发现当前对象是String，而需要的类型是User，就会使用该PropertyEditor来做类型转化
	propertyEditorMap.put(User.class, StringToUserPropertyEditor.class);
	customEditorConfigurer.setCustomEditors(propertyEditorMap);
	return customEditorConfigurer;
}
```

- 假设现有如下 bean
  - user 属性就能正常的完成属性赋值

```java
@Component
public class UserService {

	@Value("xxx")
	private User user;

	public void test() {
		System.out.println(user);
	}

}
```

### ConversionService

- Spring中提供的类型转化服务，比 PropertyEditor 更强大

```java
public class StringToUserConverter implements ConditionalGenericConverter {

	@Override
	public boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType) {
		return sourceType.getType().equals(String.class) && targetType.getType().equals(User.class);
	}

	@Override
	public Set<ConvertiblePair> getConvertibleTypes() {
		return Collections.singleton(new ConvertiblePair(String.class, User.class));
	}

	@Override
	public Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType) {
		User user = new User();
		user.setName((String)source);
		return user;
	}
}
```

```java
DefaultConversionService conversionService = new DefaultConversionService();
conversionService.addConverter(new StringToUserConverter());
User value = conversionService.convert("1", User.class);
System.out.println(value);
```

#### 向Spring注册ConversionService

```java
@Bean
public ConversionServiceFactoryBean conversionService() {
	ConversionServiceFactoryBean conversionServiceFactoryBean = new ConversionServiceFactoryBean();
	conversionServiceFactoryBean.setConverters(Collections.singleton(new StringToUserConverter()));

	return conversionServiceFactoryBean;
}
```

### TypeConvert

- 整合了 PropertyEditor 和 ConversionService 的功能
- 是Spring内部用的

```java
SimpleTypeConverter typeConverter = new SimpleTypeConverter();
typeConverter.registerCustomEditor(User.class, new StringToUserPropertyEditor());
//typeConverter.setConversionService(conversionService);
User value = typeConverter.convertIfNecessary("1", User.class);
System.out.println(value);
```

## OrderComparator

- Spring 提供的一种比较器，两种方式
  - @Order注解
  - 实现Ordered接口

- 举例：

```java
public class A implements Ordered {

	@Override
	public int getOrder() {
		return 3;
	}

	@Override
	public String toString() {
		return this.getClass().getSimpleName();
	}
}
```

```java
public class B implements Ordered {

	@Override
	public int getOrder() {
		return 2;
	}

	@Override
	public String toString() {
		return this.getClass().getSimpleName();
	}
}
```

```java
public class Main {

	public static void main(String[] args) {
		A a = new A(); // order=3
		B b = new B(); // order=2

		OrderComparator comparator = new OrderComparator();
		System.out.println(comparator.compare(a, b));  // 1

		List list = new ArrayList<>();
		list.add(a);
		list.add(b);

		// 按order值升序排序
		list.sort(comparator);

		System.out.println(list);  // B，A
	}
}
```

- OrderComparator 的子类 `AnnotationAwareOrderComparator` 支持 @Order

```java
@Order(3)
public class A {

	@Override
	public String toString() {
		return this.getClass().getSimpleName();
	}

}
```

```java
@Order(2)
public class B {

	@Override
	public String toString() {
		return this.getClass().getSimpleName();
	}

}
```

```java
public class Main {

	public static void main(String[] args) {
		A a = new A(); // order=3
		B b = new B(); // order=2

		AnnotationAwareOrderComparator comparator = new AnnotationAwareOrderComparator();
		System.out.println(comparator.compare(a, b)); // 1

		List list = new ArrayList<>();
		list.add(a);
		list.add(b);

		// 按order值升序排序
		list.sort(comparator);

		System.out.println(list); // B，A
	}
}
```

## BeanPostProcessor

- bean后置处理器，可以定义一个或多个

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

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if ("userService".equals(beanName)) {
			System.out.println("初始化后");
		}

		return bean;
	}
}
```

- 可以通过定义 BeanPostProcessor 来干涉 Spring 创建 Bean 的过程
  - 在 bean 初始化之前
  - 在 bean 初始化之后

## BeanFactoryPostProcessor

- beanFactory 后置处理器

```java
@Component
public class ZhouyuBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		System.out.println("加工beanFactory");
	}
}
```

## FactoryBean

- 自己创造一个bean，不经过传统的bean生命周期

```java
@Component
public class ZhouyuFactoryBean implements FactoryBean {

	@Override
	public Object getObject() throws Exception {
		UserService userService = new UserService();

		return userService;
	}

	@Override
	public Class<?> getObjectType() {
		return UserService.class;
	}
}
```

- 通过实现 FactoryBean 接口，这个类也会成为一个 bean
  - 但只会经过 `初始化后`
  - 其他bean生命周期都不经过，比如 `依赖注入`

## ExcludeFilter和IncludeFilter

- Spring扫描过程中用来过滤的
  - ExcludeFilter：排除过滤器
  - IncludeFilter：包含过滤器
- 案例1：
  - 扫描 com.zhouyu 保下的所有类，但是排除 UserService（不管怎样都不会成为bean）

```java
@ComponentScan(value = "com.zhouyu",
		excludeFilters = {@ComponentScan.Filter(
            	type = FilterType.ASSIGNABLE_TYPE, 
            	classes = UserService.class)}.)
public class AppConfig {
}
```

- 案例2：
  - 包含 UserService（不管怎样都会成为bean）

```java
@ComponentScan(value = "com.zhouyu",
		includeFilters = {@ComponentScan.Filter(
            	type = FilterType.ASSIGNABLE_TYPE, 
            	classes = UserService.class)})
public class AppConfig {
}
```

- Spring扫描逻辑中，默认会添加一个 AnnotationTypeFilter 给 includeFilters
  - 表示默认情况下Spring认为扫描到的类上，有 @Component 注解的就是 bean

### FilterType

- 类型分为如下几种

#### ANNOTATION

- 表示是否包含某个注解

#### ASSIGNABLE_TYPE

- 表示是否包含某个类

#### ASPECTJ

- 表示是否符合某个Aspectj 表达式

#### REGEX

- 表示是否符合某个正则表达式

#### CUSTOM

- 自定义

## MetadataReader、ClassMetadata、AnnotationMetadata

- Spring中，需要去解析类的元数据
  - 类名
  - 类中方法
  - 类上注解
  - ...
- 所以，Spring对类的元数据做了抽象，并提供了一些工具类

### MetadataReader

- 类的元数据读取器
  - 默认实现类为：`SimpleMetadataReader`
  - SimpleMetadataReader 解析类时，使用的`ASM`技术
- 案例：

```java
public class Test {

	public static void main(String[] args) throws IOException {
		SimpleMetadataReaderFactory simpleMetadataReaderFactory = new SimpleMetadataReaderFactory();
		
        // 构造一个MetadataReader
        MetadataReader metadataReader = simpleMetadataReaderFactory.getMetadataReader("com.zhouyu.service.UserService");
		
        // 得到一个ClassMetadata，并获取了类名
        ClassMetadata classMetadata = metadataReader.getClassMetadata();
	
        System.out.println(classMetadata.getClassName());
        
        // 获取一个AnnotationMetadata，并获取类上的注解信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		for (String annotationType : annotationMetadata.getAnnotationTypes()) {
			System.out.println(annotationType);
		}

	}
}
```

