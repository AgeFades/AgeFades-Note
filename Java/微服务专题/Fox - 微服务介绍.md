[TOC]

# Fox - 微服务介绍

## 资料

[微服务定义英文原文](https://martinfowler.com/articles/microservices.html)

[微服务定义中文翻译](http://blog.cuicc.com/blog/2015/07/22/microservices/#%E4%BE%A7%E8%BE%B9%E6%A0%8F%EF%BC%9A%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%92%8Csoa)

[SpringCloud技术栈脑图](https://www.processon.com/view/link/60519545f346fb348a97c9d5#map)

[SpringCloud Alibaba版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)

[SpringCloud官网](https://spring.io/projects/spring-cloud)

[SpringCloud中文文档](https://www.springcloud.cc/)

[SpringCloud中国社区](http://springcloud.cn/)

## 单机架构 VS 微服务架构

### 单机架构

#### 简介

- 一个包（例如: 一个SpringBoot Jar包）包含应用的所有功能，这就叫单体应用
- 架构单体应用的方法论，则称之为 **单体应用架构**

#### 示意图

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCEf2ba3ae24a610452391228e917ad454f/15609)

#### 优点

- 架构简单、没有复杂问题需要解决
- 开发、测试、部署简单

#### 缺点

- 随着业务迭代，代码量越来越多、越来越复杂，团队成员水平参差不齐，每次修改代码都会心惊胆战
- 部署慢，可能改了一行代码，需要部署一个200W行代码的项目（15分钟）
- 扩展成本高，无法针对单个业务功能模块进行扩展
- 阻碍技术更新，后期不敢更换最早期的技术架构

### 微服务架构

#### 简介

- 微服务核心：
  - 根据业务将单体应用拆分为一个一个的服务，彻底解耦
  - 每个服务提供独自的业务功能
  - 每个服务都能单独部署，甚至拥有自己的数据库

#### 特点

1. 独立部署，灵活扩展
2. 资源的有效隔离
3. 团队架构组织的调整

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCE99e113fda0f58276a73f752e45434292/15603)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCE99e113fda0f58276a73f752e45434292/15603)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/5E95830715C34B4AA67CE81E3CE567E5/15605)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/58FD8C69106A47F68332945B6858444C/15604)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/5D9D2EDC6BA7416AA1E2248D852ABB5A/15602)

#### 与SOA的差异

##### SOA

- 强调异构系统间的通信和解耦

##### 微服务

- 强调系统按业务边界做细粒度的拆分和部署

## Spring Cloud 微服务技术栈

### 介绍

- 提供 **快速构建微服务系统** 中的 一些常见模式的工具
  - 服务发现
  - 配置管理
  - 服务熔断
  - 网关路由
  - 消息总线
  - ...

### 图片

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCEcb3a861b4973141acbd6ce040f24d3cd/15608)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCE27faced521b89f710212f3b279554fd8/15598)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCE06d693a542656574baa78d287d1cd24e/15606)

![](https://note.youdao.com/yws/public/resource/f3687da5f73873e40890ee5323ea440e/xmlnote/WEBRESOURCEedf8e2caadc840f9cf9260a6ecc8283e/15607)

