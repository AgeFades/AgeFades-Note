# 慕课 - Spring Cloud Alibaba

## 课程导学

### 什么是 Spring Cloud Alibaba

```shell
# 阿里巴巴结合自身微服务实践，开源的微服务全家桶。

# Spring Cloud 第二代的标准实现。
```

### 项目地址

[Spring Cloud Alibaba]: https://github.com/alibaba/spring-cloud-alibaba

### Spring Cloud Alibaba 真实应用场景

```shell
# 大型复杂的系统，例如大型电商系统

# 高并发系统，例如大型门户，秒杀系统

# 需求不明确，且变更很快的系统，例如创业公司业务系统
```

### Spring Cloud Alibaba 和 Spring Cloud 的区别和联系

```shell
# Spring Cloud Alibaba 是 Spring Cloud 的子项目

# 总结来说
	# 组件性能更强
	# 良好的可视化界面
	# 搭建简单，学习曲线低
	# 文档丰富并且是中文
```

| Spring Cloud                      | 状态                                           | SpringCloud Alibaba                                       | 状态                                               |
| --------------------------------- | ---------------------------------------------- | --------------------------------------------------------- | -------------------------------------------------- |
| Eureka                            | 2.0孵化失败                                    | Nacos Discovery                                           | 性能强劲，感知更快                                 |
| Ribbon                            | 进入维护状态，预计2020年1月停止维护            | 新的标准已形成: spring-cloud-loadbalancer，但暂无参考实现 | Spring Cloud Hoxton 才会孵化出替代品               |
| Hystrix/Hystrix Dashboard/Turbine | 同上                                           | Sentinel                                                  | 可视化配置，上手更简单                             |
| Zuul                              | 同上                                           | Spring Cloud Gateway                                      | 性能是 Zuul 的1.6倍                                |
| Spring Cloud Config               | 搭建复杂，约定多，设计繁重，没有界面，难以上手 | Nacos Config                                              | 搭建简单，有可视化界面，配置管理更高效，学习曲线低 |

### 将学到什么

```shell
# Spring Cloud Alibaba 核心组件的用法以及实现原理

# Spring Cloud Alibaba 结合微信小程序从 "0" 学习真正开发中的使用

# 实际工作中如何避免采坑，正确的思考问题方式

# Spring Cloud Alibaba 的进阶: 代码的优化和改善，微服务监控
```

### 课程进阶

```shell
# 如何提升团队的代码质量

# 如何改善代码结构设计

# 编码技巧

# 心得总结

# 借助监控工具定位问题、解决问题
```

### 课程思路

```shell
# 先对小程序进行业务建模并拆分成微服务

# 编写代码

# 分析现有架构问题

# 引入微服务组件

# 优化重构

# 总结完善
```

### Spring Cloud Alibaba 的重要组件精讲

```shell
# 服务发现 Nacos
	# 服务发现原理剖析
	# Nacos Server/Client
	# 高可用 Nacos 搭建
	
# 实现负载均衡 Ribbon
	# 负载均衡的常见模式
	# RestTemplate 整合 Ribbon
	# Ribbon 配置自定义
	# 如何扩展 Ribbon
	
# 声明式 HTTP 客户端 Feign
	# 如何使用 Feign
	# Feign 配置自定义
	# 如何扩展 Feign
	
# 服务容错 Sentinel
	# 服务容错原理
	# Sentinel
	# Sentinel Dashboard
	# Sentinel 核心原理分析
	
# 消息驱动 RocketMQ
	# Spring Cloud Stream
	# 实现异步消息推送与消费
	
# API网关 Gateway
	# 整合 Gateway
	# 三大核心
	# 聚合微服务请求
	
# 用户认证与授权
	# 认证授权的常见方案
	# 改造 Gateway
	# 扩展 Feign
	
# 配置管理 Nacos
	# 配置如何管理
	# 配置动态刷新
	# 配置管理的最佳实践
	
# 调用链监控 Sleuth
	# 调用链监控原理剖析
	# Sleuth 使用
	# Zipkin 使用
```

### 项目环境搭建

#### 课上用到的软件

```shell
# JDK8

# MySQL

# Maven 安装与配置

# IDEA 配置与快捷键技巧
```

#### Maven 安装与配置

##### 下载地址

[Maven]: http://maven.apache.org/download.cgi

## Spring Boot 基础

### Spring Boot 是什么？能做什么？

```shell
# Spring Boot 是一个快速开发的脚手架

# 作用:
	# 快速创建独立的、生产级的基于Spring 的应用程序
	
# 特性:
	# 无需部署 WAR 文件
	# 提供 starter 简化配置
	# 尽可能自动配置 Spring 以及第三方库
	# 提供 "生产就绪" 功能，例如指标、健康检查、外部配置等
	# 无代码生成 & 无 XML
```

### Spring Boot 必知必会

```shell
# 快速创建应用
	# 略...

# 应用组成分析
	# 依赖 pom.xml
	# 启动类注解
	# 配合 application.yml
	# 静态文件 static 目录
	# 模板文件 template 目录

# 开发三板斧
	# 加依赖
	# 写注解
	# 写配置

# Spring Boot Actuator 监控
	# 略...

# 配置管理
	# 支持的配置格式
		# properties
		# yml
	# 配置管理常用方式
		# 配置文件
		# 环境变量
		# 外部配置文件
		# 命令行参数

# Profile
	# 如何实现不同环境不同配置
```

