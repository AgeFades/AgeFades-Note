# 周阳 - SpringCloud

## Eureka

### Eureka 基础知识

#### 什么是服务治理

```shell
# 在传统的 rpc 远程调用框架中，管理服务之间的依赖关系比较复杂，所以需要使用服务治理。

# 管理服务之间依赖关系，
	# 实现服务发现与注册、调用、负载均衡、容错等。
```

#### 什么是服务注册与发现

```shell
# 在服务注册与发现中，有一个注册中心。

# 当服务器启动的时候，会把当前自己服务器的信息，
	# 比如 服务地址、通讯地址 等以别名方式注册到注册中心上，并维持心跳连接。	
	
# 另一方以该别名的方式去注册中心获取到实际的服务通讯地址，然后再实现本地 RPC 调用。
```

![UTOOLS1583461034114.png](http://yanxuan.nosdn.127.net/9e34edec94218185cc5059450cb65ca3.png)

#### Eureka 两组件

```shell
# Eureka Server
	# 各个微服务节点通过配置启动后，会在 EurekaServer 中进行注册，
	# EurekaServer 的服务注册表中将会存储所有可用服务节点的信息。
	
# Eureka Client
	# 一个 Java 客户端，用于简化 Eureka Server 的交互，
	# 同时具备一个内置的、使用轮询(round-robin) 负载算法的负载均衡器。
	# 应用启动后，将会向 Eureka Server 发送心跳(默认周期为30秒)，
	# 如果 Server 在多个心跳周期内没有接收到某个节点的心跳，
	# Server 将会从服务注册表中把这个服务节点移除（默认90秒）
```

### 单机 Eureka 构建步骤

#### Eureka Server

```shell
# 建 Module

# 改 POM

# 写 YML

# 主启动
```

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```yaml
server:
	port: 7001 # 服务端口号

eureka:
	instance:
		hostname: localhost # eureka 服务端的实例名称
	client:
		register-with-eureka: false # 不向注册中心注册自己
		fetch-registry: false # false 表示不需要去检索服务
		service-url:
			# 与 Eureka Server 交互的地址查询服务和注册服务都需要依赖这个地址
			defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

```java
// 主启动类上加注解
@EnableEurekaServer
```

#### Eureka Client

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
	applicaiton:
		name: payment # 向注册中心注册的服务名，非常重要

eureka:
	client:
		register-with-eureka: true # 将自己注册进 EurekaServer
		fetch-registry: true # 默认为 true，集群时必须设置为 true 才能配合 ribbon 负载均衡
		service-url:
			# 与 Eureka Server 交互的地址查询服务和注册服务都需要依赖这个地址
			defaultZone: http://localhost:7001/eureka # 服务中心的注册地址
```

```java
// 主启动类上加注解
@EnableEurekaClient
```

#### Eureka 的自我保护机制

![UTOOLS1583463177265.png](http://yanxuan.nosdn.127.net/8ec3afd579eeadcc8ba219f6c0673fd7.png)

### 集群 Eureka 构建步骤

```

```

