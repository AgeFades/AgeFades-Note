[TOC]

# 徐庶 - Spring监听器原理学习

## 使用Spring事件

- Spring事件体系包括三个组件
  - `事件`
  - `事件监听器`
  - `事件广播器`
- 原理就是 `观察者模式`

### 事件

#### Spring内置事件

- 内置事件 由 系统内部 进行发布，只需注入监听器

![](https://note.youdao.com/yws/public/resource/42d15ea5ab2072b4a354441ce9080eb9/xmlnote/39FCB85E7FCE4C3FB0E542D4A8751C12/7078)

##### ContextRefreshedEvent

- `ApplicationContext.refresh()` IOC刷新完毕、所有Bean准备完毕时，发布该事件
  - IOC容器支持热部署时，该事件会被发布多次（对应多次 refresh()）
  - 如: `XmlWebApplicationContext`

##### ContextStartedEvent

- 容器启动时发布，即调用 start() 方法
- 已启用意味着所有的 `Lifecycle Bean` 都已显式接收到 start 信号

##### ContextStoppedEvent

- 容器停止时发布，即调用 stop() 方法
- 已停止意味着所有的 `Lifecycle Bean` 都已显式接收到 stop 信号
- 关闭的容器可以通过 start() 方法重启

##### ContextClosedEvent

- 容器关闭时发布，即调用 close() 方法
- 意味所有Bean都已被销毁
- 关闭的容器不能被重启或 refresh()

##### RequestHandledEvent

- 在 SpringMVC 的 DispatcherServlet 时有效
- 当一个请求被处理完成时，发布该事件

#### 自定义事件

##### 基于继承

```java
public class ExtendsEvent extends ApplicationEvent {
  private String name;
  
  public ExtendsEvent(Object source, String name) {
    super(source);
    this.name = name;
  }
  
  public String getName() {
    return name;
  }
}
```

#### 自定义监听器

##### 基于接口

```java
@Component
public class InterfaceListener implements ApplicationListener<OrderEvent> {
  
  @Override
  public void onApplicationEvent(ExtendsEvent event) {
    System.out.println(event);
  }
  
}
```

##### 基于注解

```java
@Component
public class AnnotationListener {
  
  @EventListener(OrderEvent.class)
  public void onApplicationEvent(ExtendsEvent event) {
    System.out.println(event);
  }
  
}
```

#### 事件发布操作

```java
applicationContext.publishEvent(new ExtendsEvent(this, "张三"));
```

![](https://note.youdao.com/yws/public/resource/42d15ea5ab2072b4a354441ce9080eb9/xmlnote/E3E9ABECD8EF4425B9F8ED5D9F877BE9/7108)

### 问题

- 一个事件可以有多个监听器吗？
  - 可以
- 监听器必须基于实现接口吗？
  - 不是，可以用注解 @EventListener
- 发布、监听，两个操作是同步的吗？
  - 是，所以有事务的话，监听操作也在同个事务内
- 监听可以异步处理吗？
  - 可以，加 @Async

## 事件发布器

![](https://note.youdao.com/yws/public/resource/42d15ea5ab2072b4a354441ce9080eb9/xmlnote/D6A3C89D36414981BC1B5C9B70FBC026/5791)

- `applicationContext.publishEvent()` 实际将 `事件` 发送到 `EventMultiCaster` (事件发布器)
- 再由 EventMultiCaster 根据事件类型，转发至注册的对应 Listener

### 源码流程

![](https://note.youdao.com/yws/public/resource/42d15ea5ab2072b4a354441ce9080eb9/xmlnote/38AAA0A1D11F42ED9051151F03F713EB/7154)

- `AbstractApplicationContext.refresh()` 中，`initApplicationEveenntMulticaster()` 创建了事件发布器、监听注册表

## 备注

- 因为Spring监听器实际运用不多、原理较为简单，记录至此。