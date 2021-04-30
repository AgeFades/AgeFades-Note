[TOC]

# 徐庶 - Spring中的设计模式

## 简单工厂模式

### 实现方式

- 通过 Spring 中的 `BeanFactory` 实现
- getBean() 根据传入一个唯一标识来获得相应 Bean 对象

### 实质

- 由一个 `工厂类` 根据 `传入的参数`，`动态 `决定应该创建哪一个 `产品类`

### 实现原理

#### Bean容器的启动阶段

- 读取 bean 配置，将 bean 元素分别转换成一个 BeanDefinition 对象
- 通过 BeanDefiniitionRegistry 将这些 BeanDefinition 注册到 BeanFactory 中
  - 保存在 BeanFactory 中的一个 ConcurrentHashMap 中
- 注册完后，可以通过实现 BeanFactoryPostProcessor 在此处插入开发自己定义的代码
  - 例: `PropertyPlaceholderConfigurer`
  - 即一般在配置 数据库 的 dataSorce 时，使用到的占位符的值，就是它注入进去的

#### Bean的实例化阶段

- 主要通过反射对 Bean 进行实例化，这个阶段有许多扩展点
- 各种 Aware 接口
  - 例: `BeanFactoryAware`
  - 对实现这些 Aware 接口的 Bean，实例化时，Spring 会注入对应的 BeanFactory 实例
- `BeanPostProcessor` 接口
  - 对实现了 BeanPostProcessor 接口的 Bean，实例化时，Spring 会调用对应接口的实现方法
- `InitializingBean` 接口
  - 对实现了 InitializingBean 接口的Bean，实例化时，Spring 会调用对应接口的实现方法
- `DisposableBean` 接口
  - 对实现了 `BeanPostProcessor` 接口的Bean，死亡时，Spring 会调用对应接口的实现方法

### 设计意义

- 解耦
- Bean的可扩展性

## 工厂方法

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247487170&idx=2&sn=34b135090c064c1ca202d629a4beab52&chksm=ebd631eedca1b8f85e444f7544c2cbe696b253fe3916f17dbcfbab49b6b74653126c230fdc39&scene=21#wechat_redirect)

### 实现方式

- FactoryBean 接口

### 实现原理

- 实现了 FactoryBean 接口的 Bean 叫做 Factory
- 特点: Spring 会在 getBean() 获得该 Bean 时，自动调用该 Bean 的 getObject()
- 所以返回不是 Factory 这个 Bean，而是其 getObject() 的返回值

### 代码示例

```xml
<!-- SqlSessionFactory -->
<bean class="org.mybatis.spring.SqlSessionFactoryBean" id="sqlSessionFactory">
  <!-- 指定Spring中的数据源 -->
  <property name="dataSource" ref="dataSource"></property>
  <property name="configLocation" value="classpath:mybatis-config.xml"></property>
  <property name="mapperLocations" ref="classpath:com/demo/mapper/*.xml"></property>
</bean>
```

- 代码中的 SqlSessionFactoryBean 因为实现了 FactoryBean
- 所有返回不是 SqlSessionFactoryBean 的实例，而是 SqlSessionFactoryBean.getObject() 的返回值

## 单例模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247485826&idx=2&sn=e21d6188ea07a992f1eb9a6671ae7485&chksm=ebd636aedca1bfb8ff0ad69343ab40e87cd65ec41e2dfc54d761e97e9058effe6bd2eac28486&scene=21#wechat_redirect)

### 简介

- Spring 依赖注入 Bean 实例默认都是单例的
- 因为 Spring 依赖注入（包括 lazy-init 方式）都是发生在 AbstractBeanFactory.getBean() 中
- 而 getBean() 中 doGetBean() 是调用 getSingleton 进行 bean 的构建

### 代码

```java
/**
 * @param allowEarlyReference 标识是否允许早期依赖
 */
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 检查缓存中是否存在实例
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
      // 如果实例为空，锁定全局变量进行处理
			synchronized (this.singletonObjects) {
        // 如果该 Bean 正在加载，则不处理
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
          // 当某些方法需要提前初始化时，则会当调用 addSingleFactory()
          // 将对应的 ObjectFactory 初始化策略存储在 SingletionFactories
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
            // 调用预选设定的 getObject() 方法
						singletonObject = singletonFactory.getObject();
            // 记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```

### 图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608622882130.png)

### 总结

- 单例模式的定义:
  - 保证一个类仅有一个实例，并提供一个访问它的全局访问点
- Spring对单例的实现
  - Spring提供了全局的访问点 BeanFactory
  - 但没有从构造器级别去控制单例，因为 Spring 管理任意 Java 对象

## 适配器模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247485849&idx=2&sn=79922e3fe8278d01e3fab1870ed824bc&chksm=ebd636b5dca1bfa3c83c3d2d740e25553bc1ba686b2ea6de46acc14d9ea547eb636e24ac7db0&scene=21#wechat_redirect)

### 实现方式

- SpringMVC 中的适配器 HandlerAdapter

### 实现原理

- HandlerAdapter 根据 Handler 规则执行不同的 Handler

### 实现过程

1. DispatcherServlet 根据 HandlerMapping 返回的 Handler
2. 向 HandlerAdapter 发起请求，处理 Handler
3. HandlerAdapter 根据规则找到对应 Handler 并让其执行
4. 执行完毕后，返回 ModelAndView

### 实现意义

- Spring定义该适配接口，使每种 controller 都有一种对应 适配器实现类

## 装饰器模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247486377&idx=2&sn=e88370d32c36b19ac89189341cbaf03b&chksm=ebd63485dca1bd93fd46ce901b8ed5adaa0f1f5db15b8a19902bef66a05bd38ed420e26f7f5e&scene=21#wechat_redirect)

### 实现方式

- Spring中装饰器模式，在类名上有两种表现
  - 含 Wrapper
  - 含 Decorator

### 实现原理

- 动态给一个对象添加一些额外的职责
- 就 增加功能 来说，Decorator 模式相比 生成子类 更加灵活

## 代理模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247487054&idx=2&sn=489695986c038525e25c017c217b72fb&chksm=ebd63162dca1b874edcaa30680e1da4d3a02c9b0011cb5c60c22d2b9ebea9169022813810bd5&scene=21#wechat_redirect)

### 实现方式

- AOP底层，就是 动态代理 模式的实现
- `动态代理`：
  - 内存中构建，不需要手动编写代理类
- `静态代理`:
  - 需要手动编写代理类，代理类引用被代理对象

## 观察者模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247486897&idx=2&sn=927e4dd95695163fa250447ea88e12e8&chksm=ebd6329ddca1bb8bebcd94bc31396ae55c6f28e0ec2057203605aedf9b8cb4a7243c1202e3c5&scene=21#wechat_redirect)

### 实现方式

- Spring的 事件驱动模型 使用的就是 观察者模式
- Spring中 Observer 模式常用于 Listener 的实现

### 具体实现

- 事件机制的实现需要三个部分
  - `事件源`
  - `事件`
  - `事件监听器`

#### 事件

- `ApplicationEvent` 继承自 JDK 的 EventObject
- 所有事件都需要继承 ApplicationEvent，并通过构造器 source 获得事件源
- 实现类 `ApplicationContextEvent` 表示 ApplicationContext 的容器事件

```java
public abstract class ApplicationEvent extends EventObject {
    private static final long serialVersionUID = 7099057708183571937L;
    private final long timestamp = System.currentTimeMillis();

    public ApplicationEvent(Object source) {
        super(source);
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

#### 事件监听器

- `ApplicationListener` 继承自 JDK 的 EventListener
- 所有监听器都要实现该接口, 在 onApplicationEvent() 中接收 事件 参数
- 方法体中，通过对不同 Event 判断进行相应处理
- 当事件触发时，所有监听器都会收到消息

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E var1);
}
```

#### 事件源

- `ApplicationContext` 是 Spring 中的全局容器，翻译为 应用上下文
- 实现了 ApplicationEventPublisher 接口
- 负责读取 Bean 的配置文件、管理Bean的加载、维护Bean之间的依赖关系
- 是负责Bean的整个生命周期，即所说 `IOC容器`

```java
@FunctionalInterface
public interface ApplicationEventPublisher {
    default void publishEvent(ApplicationEvent event) {
        this.publishEvent((Object)event);
    }

    void publishEvent(Object var1);
}
```

#### ApplicationEventMulticaster

- ApplicationEventPublisher 实际调用该类
- 事件广播器，将 ApplicationContext 发布的 Event 广播给所有的监听器

## 策略模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247485903&idx=1&sn=172401bccf663455fd90c55aa957db18&chksm=ebd636e3dca1bff544c671c9f8de3a6e8d40049ab7cea3d750d03ed00ef6e6c4fae376333de1&scene=21#wechat_redirect)

### 实现方式

- Spring Resource 接口，用于访问底层资源

### Resource核心API

#### getInputStream()

- 定位并打开资源，获取资源对应的输入流
- 每次调用都会返回新的输入流，调用者必须负责关闭输入流

#### exists()

- 返回 Resouce 所指向的资源是否存在

#### isOpen()

- 返回资源文件是否打开
- 如果资源文件不能多次读取，每次读取结束应该显示关闭，防止资源泄漏

#### getDescription()

- 返回资源的描述信息
- 通常用于资源处理出错时输出该信息

#### getFile()

- 返回资源对应的 File 对象

#### getURL()

- 返回资源对应的 URL 对象

### Resource的策略模式

- Resource 接口本身没有提供访问任何底层资源的实现逻辑
- 针对不同的底层资源，Spring 提供不同的 Resource 实现类
- 例如:
  - `UrlResource`： 访问网络资源的实现类
  - `ClassPathResource`：访问类加载路径里资源的实现类
  - `FileSystemResource`：访问文件系统里资源的实现类
  - `ServletContextResource`：访问相对于 ServletContext 路径的资源
  - `InputStreamResource`：访问输入流资源的实现类
  - `ByteArrayResource`：访问字节数组资源的实现类

## 模板方法模式

[参考资料](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247486148&idx=1&sn=601fa38aee0aa27137341ce9a2624fec&chksm=ebd635e8dca1bcfe8da575478244414d13620010cd0d9823f423af8d2457ad1bb65a17d6a940&scene=21#wechat_redirect)

### 实现方式

- Spring 中 Template 都是模板方式的实现
- 例如:
  - JdbcTemplate
  - redisTemplate