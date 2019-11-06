# 李卫民 - SpringCloud Alibaba

## 简介

```shell
# 阿里巴巴开源的微服务解决方案

# 微服务架构，更好的进行分布式系统开发
	# 拆分单体应用，将一个应用拆分成多个服务，每个服务都是单独可以运行的项目
```

### 分布式系统开发的问题

```shell
# 这么多服务，客户端如何访问？

# 这么多服务，服务之间如何通信？

# 这么多服务，如何治理？

# 服务挂了，怎么办?
```

### 解决方案

#### Spring Cloud Netflix

```shell
# API 网关 -> Zuul

# 服务注册与发现 -> Eureka

# 服务通信 -> Feign<Http Client,基于 Http 的通信协议，同步并阻塞>

# 熔断机制 -> Hystrix

# 2018.12.12 宣布项目进入维护模式
```

#### Apache Dubbo Zookeeper

```shell
# Dubbo 是一个高性能的 Java RPC 通信框架

# 利用 Zookeeper 作为服务的发现与注册中心

# 其他功能需要糅合别的组件
```

#### Spring Cloud Alibaba

```shell
# 
```

