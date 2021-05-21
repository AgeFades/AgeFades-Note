# Fox - Feign

## Java中实现接口调用的几种方式

### HttpClient

- Apache 子项目，相比 JDK 自带的 URLConnection，提升了易用性和灵活性

### OkHttp

- Square 公司开源项目，安卓端最火的轻量级框架
- 简洁API、高效性能，支持多种协议（HTTP/2 和 SPDY）

### HttpURLConnection

- JDK 自带，使用较为复杂

### RestTemplate

- Spring 提供基于 Rest 服务的访问客户端

## Feign

### 简介

- Nextflix 开发的声明式、模板化的 Http 客户端
  - 参考了 Retrofix、JAXRS-2.0、WebSocket
- 支持多种注解，例如：
  - Feign自带注解
  - JAX-RS 注解
  - ......
- Spring Cloud OpenFeign 进行了增强
  - 支持 Spring MVC 注解
    - 整合 Ribbon 和 Eureka

### 优势

- 使 Http 远程调用 像 本地方法调用 一样的开发体验（RPC的设计思想）
- 像 Dubbo 中 Consumer 直接调用接口方法来调用 Provider
- 无需关注 Http 请求构造、数据解析、网络交互等细节

### 设计架构

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/7CF1DB80DA3F42F79E808DA57F76CD34/15690)

### 集成