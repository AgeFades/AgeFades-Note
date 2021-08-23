[TOC]

# Fox - Gateway

## 资料

[官方文档](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#gateway-request-predicates-factories)

## 简介

- 网关 -> 流量的入口，通常负责：
  - 路由转发
  - 权限校验
  - 限流
  - ...
- Spring Cloud Gateway 用于取代 Zuul
  - 基于 WebFlux、Netty、Reactor 实现的响应式API网关
  - 不是传统的 Serlvet 容器，也不能构建成 war 包

### 核心概念

#### 路由（route）

- 网关最基础部分，路由信息包括
  - 一个ID
  - 一个目的URL
  - 一组断言工厂
  - 一组Filter
- 断言为真时，说明请求的URL和配置的路由相匹配，可以进行转发

#### 断言（predicates）

- 使用断言函数对请求进行断言，可根据 Http Requst 中的任何信息做断言

#### 过滤器（Filter）

- Gateway 有两种 Filter：
  - Gateway Filter
  - Global Filter
- Filter 可对请求和响应进行处理

### 工作原理

- 跟 Zuul 差不多，最大的区别就是：
  - Gateway 的 Filter 只有 Pre 和 Post 两种
- 客户端发起请求 -> 
  - Gateway -> 
  - 匹配请求与网关中定义的路由 ->
  - 成功则 经过过滤器链 ->
  - 执行所有 Pre Filter ->
  - 发送代理请求、获取返回结果 ->
  - 执行 Post Filter

![](https://note.youdao.com/yws/public/resource/e6a0c6530154a553fd11593f56b78c9a/xmlnote/B5EE5146728D46178B20BE6818FB9963/13816)

## 快速开始

### 依赖

- 注意排除 SpringMvc 的依赖（容器依赖冲突）

```xml
<!-- gateway网关 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<!-- nacos服务注册与发现 -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

### 配置

```yaml
server:
  port: 8888
  
spring:
  application:
    name: mall-gateway
  #	配置nacos注册中心地址
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848

    gateway:
      discovery:
        locator:
          # 默认为false，设为true开启通过微服务创建路由的功能，即可以通过微服务名访问服务
          enabled: true
      # 是否开启网关    
      enabled: true 
```

### 测试

![](https://note.youdao.com/yws/public/resource/e6a0c6530154a553fd11593f56b78c9a/xmlnote/69A047A755E94EE7B6BED46BF9489FAD/13813)

## 路由断言工厂配置

[参考链接](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/SpringCloud%20Gateway/%E6%96%B9%E5%BF%97%E9%B9%8F%20-%20SpringCloud%20Gateway.md)

## 过滤器工厂配置

[参考链接](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/SpringCloud%20Gateway/%E6%96%B9%E5%BF%97%E9%B9%8F%20-%20SpringCloud%20Gateway.md)

### 全局鉴权Filter的一种思路

```java
@Component
@Order(-1)
@Slf4j
public class CheckAuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        //校验请求头中的token
        List<String> token = exchange.getRequest().getHeaders().get("token");
        log.info("token:"+ token);
        if (token.isEmpty()){
            return null;
        }
        return chain.filter(exchange);
    }
}
```

### 全局处理Ip白名单的一种思路

```java
@Component
public class CheckIPFilter implements GlobalFilter, Ordered {

    @Override
    public int getOrder() {
        return 0;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        HttpHeaders headers = exchange.getRequest().getHeaders();
        //模拟对 IP 的访问限制，即不在 IP 白名单中就不能调用的需求
        if (getIp(headers).equals("127.0.0.1")) {
            return null;
        }
        return chain.filter(exchange);
    }

    private String getIp(HttpHeaders headers) {
        return headers.getHost().getHostName();
    }
}
```

## Gateway跨域配置

[官方资料](https://docs.spring.io/spring-cloud-gateway/docs/current/reference/html/#cors-configuration)

### yml配置

```yaml
spring:
  cloud:
    gateway:
        globalcors:
          cors-configurations:
            '[/**]':
              allowedOrigins: "*"
              allowedMethods:
              - GET
              - POST
              - DELETE
              - PUT
              - OPTION
```

### Java配置

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```

## Gateway 整合 Sentinel 限流 TODO