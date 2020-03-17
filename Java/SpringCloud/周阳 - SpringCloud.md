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

```shell
# 保护模式主要哦用于一组客户端和 Eureka Server 之间存在网络分区场景下的保护。

# 一旦进入保护模式，Eureka Server 将会尝试保护其服务注册表中的信息，
	# 也就是说不会注销任何微服务。
	
# 就是收某一时刻，某个或某些微服务不可用了，Eureka 不会立即清理，依旧会保留该服务信息。

# 属于 CAP 里的 AP 分支，高可用思想。

# 当 EurekaServer 节点在短时间内丢失过多客户端时，就会进入自我保护模式。

# 主要就是应对网络异常的保护机制。
```

##### 关闭自我保护机制

```yaml
eureka:
	server:
		# 关闭自我保护机制，保证不可用服务被及时剔除
		enable-self-preservation: false
		# 客户端向服务端发送心跳的时间间隔，单位为秒，默认 30S
		eviction-interval-timer-in-seconds: 1
		# Eureka 服务端收到最后一次心跳后等待时间上限，超时将剔除服务
		lease-expireation-duration-in-seconds: 2
```

### 集群 Eureka 构建步骤

#### Eureka 集群原理说明

![UTOOLS1583476014198.png](http://yanxuan.nosdn.127.net/6d249eb1dc114446a1f573afe847d8a4.png)

```shell
# 微服务 RPC 远程服务调用最核心的就是服务的注册与发现（即: 注册中心）

# 为了防止单点故障，所以一定要搭建注册中心集群，实现负载均衡 + 故障容错

# Eureka 集群原理: 互相注册、相互守望
```

#### Eureka 集群环境构建步骤

```yaml
# 主要就是修改 application.yml
server:
	port: 7001
	
eureka:
	instance:
		hostname: eureka1.com # eureka 第一个实例所在域名
	client:
		register-with-eureka: false
		fetch-registry: false
		service-url:
			defaultZone: http://eureka2.com/eureka/ # 互相注册，eureka 第二个实例所在域名
```

```yaml
server:
	port: 7001
	
eureka:
	instance:
		hostname: eureka2.com # eureka 第一个实例所在域名
	client:
		register-with-eureka: false
		fetch-registry: false
		service-url:
			defaultZone: http://eureka1.com/eureka/ # 互相注册，eureka 第二个实例所在域名
```

#### Eureka 集群下的服务注册

```yaml
# 主要就是修改 yaml
eureka:
	client:
		register-with-eureka: true
		fetchRegistry: true
		service-url:
			defaultZone: http://eureka1.com/eureka/,http://eureka2.com/eureka
```

#### @LoadBalanced

```java
// 使用 @LoadBalanced 开启 RestTemplate 的负载均衡功能
// Ribbon 和 Eureka 整合后消费者可以直接调用服务而不用再关心地址和端口号。
@LoadBalanced
@Bean
public RestTemplate restTemplate() {
  return new RestTemplate();
}
```

### Actuator 微服务信息完善

```yaml
# 修改 yaml
eureka:
	instance-id: payment1 # 修改服务名称
	prefer-ip-address: true # 访问路径可显示 IP 地址
	client:
		register-with-eureka: true
		fetchRegistry: true
		service-url:
			defaultZone: http://eureka1.com/eureka/,http://eureka2.com/eureka	
```

### 服务发现 Discovery

```shell
# 对于注册进 eureka 里面的微服务，可以通过服务发现来获得该服务的信息。
```

```java
// 启动类上加注解
@EnableDiscovery
```

```java
// 控制器上加端点
@Resource
private DiscoveryClient discoveryClient;

@GetMapping("discovery")
public Object discovery() {
  // 获取服务中心中的服务列表
  List<String> service = discoveryClient.getService();
  service.forEach(e -> log.info("服务中心中的服务: {}", e));
  
  // 获取 xx 服务下的各实例对象
  List<ServiceInstance> instances = discoveryClient.getInstances("PAYMENT");
  instances.forEach(i -> 
                    log.info("服务ID: {}, IP: {}, 端口: {}, URI: {}", 
                            i.getServiceId(), i.getHost(), 
                            i.getPort(), i.getUri())
                   );
  
  return discoveryClient;
}
```

## Zookeeper 注册中心

### 简介

```shell
# ZK 是一个分布式协调工具，可以实现注册中心功能
```

### 服务注册进 ZK

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
```

```yaml
spring:
	cloud:
		zookeeper:
			connect-string: localhost:2181
```

```java
// 启动类上加注解
@EnabelDiscoveryClient
```

## Consul

```shell
# 略
```

