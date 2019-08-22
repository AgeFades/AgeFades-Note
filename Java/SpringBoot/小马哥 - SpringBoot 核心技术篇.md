# 小马哥 - SpringBoot 核心技术篇

## 课程导学

![UTOOLS1565749487812.png](https://i.loli.net/2019/08/14/zFdQOkSnofjZbX5.png)

## 为什么说 SpringBoot 易学难精？

### SpringBoot 易学

#### 	组件自动装配

​		规约大于配置，专注核心业务

#### 	外部化配置

​		一次构建、按需调配、导出运行

#### 	嵌入式容器

​		内置容器、无需部署、独立运行

#### 	Spring Boot Starter

​		简化依赖、按需装配、自我包含

#### 	Production-Ready

​		一站式运维、生态无缝整合

### SpringBoot 难精

#### 	组件自动装配

##### 		模式注解

##### 		@Enable 模块

##### 		条件装配

##### 		加载机制

#### 	外部化配置

##### 		Environment 抽象

##### 		生命周期

##### 		破坏性变更

#### 	嵌入式容器

##### 		Servlet Web 容器

##### 		Reactive Web 容器

#### 	SpringBoot Starter

##### 		依赖管理

##### ​		装配条件

##### 		装配顺序

#### 	Production-Ready

##### 		健康检查

##### 		数据指标

##### 		@Endpoint 管控	

### Spring Boot 与 Java EE 规范

#### 	Web

​		Servlet（JSR-315、JSR-340）

#### 	SQL

​		JDBC（JSR-221）

#### 	数据校验

​		Bean Validation（JSR 303、JSR-349）

#### 	缓存

​		Java Caching API（JSR-107）

#### 	WebSockets

​		Java API for WebSocket（JSR-356）

#### 	Web Service

​		JAX-WS（JSR-224）

#### ​	Java 管理

​		JMX（JSR 3）	

#### 	消息

​		JMS（JSR-914）

## SpringBoot 核心特性

### SpringBoot 三大特性

#### 	组件自动装配

​		Web MVC，Web Flux，JDBC 等

#### 	嵌入式 Web 容器

​		Tomcat、Jetty、Undertow

#### 	生产准备特性

​		指标、健康检查、外部化配置等

### 组件自动装配

#### 	激活

```java
@EnableAutoConfiguration // 启用组件自动装配
```

#### 	配置

```shell
/META-INF/spring.factories # KV 形式，SpringBoot 应用的组件自动装配规约
```

#### 	实现

```shell
XXXAutoConfiguration # 组件自动装配规约的实现
```

#### 	SpringBoot 应用的逻辑

```shell
# 作用在SpringBoot应用中的启动类上
# 在该注解中，包含 @EnableAutoConfiguration
@SpringBootApplication
```

### 嵌入式 Web 容器

#### 	Web Servlet

#### 	Web Reactive

​		Netty Web Server

### 生产准备特性

#### 	指标

​		/actuator/metrics

#### ​	健康指标

​		/actuator/health

#### 	外部化配置

​		/actuator/configprops	

## Web 应用

### 传统 Servlet 应用

​	Servlet 组件 : Servlet、Filter、Listener

​	Servlet 注册 : Servlet 注解、Spring Bean、RegistrationBean

​	异步非阻塞 : 异步 Servlet、非阻塞 Servlet

