# Fox - Ribbon

## 主流负载方案

- 集中式负载均衡
  - 在 **消费者** 和 **服务提供方** 中间使用独立的代理方案进行负载
    - 硬件案例: F5
    - 软件案例: Nginx

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/074F758C9974417991452EC230F43ED5/13572)

- 客户端按需负载
  - 客户端会先拿到需调用服务的实例数据集合，在发送请求前通过负载均衡算法选定某一个发送
  - 即：在客户端就进行负载均衡算法分配
    - 案例: Ribbon

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/BD4B711B137D466A81C27A487B316BE8/13568)

### 常见负载均衡算法

- 随机
  - 随缘发（用的少）
- 轮询
  - 排队轮询（默认）
- 权重
  - 配置高的机器权重大，概率大
- Hash
  - 通过客户端请求地址，进行 Hash 取模映射调度
- 最小活跃数
  - 发给压力最小的机器

## Ribbon

### 简介

- Spring Cloud Ribbon 是基于 Netflix Ribbon 实现的一套 **客户端按需负载均衡工具**
- 提供一系列的完善配置，如：
  - 超时
  - 重试
  - ...
- 通过 **Load Balancer** 获取到 服务提供的所有实例数据，基于指定规则调用服务

### 模块

- `ribbon-loadbalancer`
  - 负载均衡模块，可单独使用 或 混合使用
- `Ribbon`
  - 内置各种实现的负载均衡算法
- `ribbon-eureka`
  - 基于 Eureka 封装的模块，能快速、方便集成 Eureka
- `ribbon-transport`
  - 基于 Netty 实现多协议的支持，比如 Http、Tcp、Udp ...
- `ribbon-httpclient`
  - 基于 Apache HttpClient 封装的 Rest 客户端，集成负载均衡模块
- `ribbon-example`
  - 使用代码示例
- `ribbon-core`
  - 核心 & 通用 代码

