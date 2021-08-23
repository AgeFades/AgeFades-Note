[TOC]

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

[参考链接](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8/%E6%9C%8D%E5%8A%A1%E9%97%B4%E8%B0%83%E7%94%A8/openFeign/Feign%E9%9B%86%E6%88%90.md)

### 单独使用

#### 依赖

```xml
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-core</artifactId>
    <version>8.18.0</version>
</dependency>

<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-jackson</artifactId>
    <version>8.18.0</version>
</dependency>
```

#### 接口

```java
/**
 * 这里用的原生 Feign 注解，可以了解 openFeign 的底层实现细节
 */ 
public interface RemoteService {

    @Headers({"Content-Type: application/json","Accept: application/json"})
    @RequestLine("GET /order/findOrderByUserId/{userId}")
    public R findOrderByUserId(@Param("userId") Integer userId);

}
```

#### 调用

```java
public class FeignDemo {

    public static void main(String[] args) {

        // JSON序列化、反序列化处理请求包数据
        RemoteService service = Feign.builder()
                .encoder(new JacksonEncoder())
                .decoder(new JacksonDecoder())
                .options(new Request.Options(1000, 3500))
                .retryer(new Retryer.Default(5000, 5000, 3))
                .target(RemoteService.class, "http://localhost:8020/");

        System.out.println(service.findOrderByUserId(1));
    }
}
```

## Spring Cloud Feign 的自定义配置及使用

### 日志配置

- 在 Feign 调用时，可能遇到 Bug（接口调用失败、参数没接收到...），或者想看看调用性能时，
- 就可以对 Feign 的日志进行配置

#### 日志级别

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1621560679071.png)

##### NONE

- 不记录任何日志（默认值）
- 性能最佳，适用于生产环境

##### BASIC

- 仅记录 请求方法、URL、响应状态代码、执行时间
- 适用于生产环境追踪问题

##### HEADERS

- 在 BASIC 级别的基础上，额外记录 请求和响应的 Header

##### FULL

- 记录请求和响应的 Header、Body、元数据
- 适用于开发、测试环境定位问题

#### 全局配置

```java
// 加上 @Configuration 则全局生效
@Configuration
public class FeignConfig {
  
    /**
     * 这里可以根据项目启动环境，配置不同的日志级别
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
}
```

#### 局部配置

##### 代码配置

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/20EDD014582D426F8A5ADEF751C51E00/13333)

##### yml配置

```yaml
# 对应的源码配置类为: FeignClientConfiguration
feign:
  client:
    config:
      mall-order:  #对应微服务
        loggerLevel: FULL
```

#### 公共配置

- 需指定 Feign 包的日志级别为 debug 方可生效

```yaml
logging:
  level:
    com.tuling.mall.feigndemo.feign: debugb
```

#### BASIC测试案例

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/BFBB0C3E179B4D5581FA57E18713F244/13334)

### 契约配置

- Spring Cloud OpenFeign 默认的注解使用契约是 SpringMvcContract
- 可以通过配置这个契约来改变使用的注解

#### 修改配置

##### 代码配置

```java
@Bean
public Contract feignContract() {
  	/**
  	 * 支持 Feign 原生注解
  	 */
    return new Contract.Default();
}
```

##### yml配置

```yaml
feign:
  client:
    config:
      mall-order:  #对应微服务
        loggerLevel: FULL
        contract: feign.Contract.Default   #指定Feign原生注解契约配置
```

#### 使用

- 参考上面的 Feign 单独使用

### 拦截器配置

- 企业开发中，通常调用接口都是有权限认证的，认证数据（当前用户id等属性）传递方式通常有：
  - 参数传递
  - 请求头传递（这个好）

#### Basic认证

```java
@Configuration  // 全局配置
public class FeignConfig {
    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("fox", "123456");
    }
}
```

#### 自定义拦截器认证

##### Feign的拦截器扩展点

- 扩展点：feign.RequestInterceptor
- 每次 feign 发起 Http 调用之前，会先去执行拦截器中的逻辑
- 使用场景：
  - 统一添加 Header 信息
  - 对 Body 中的信息做 修改或替换

```java
public interface RequestInterceptor {

  /**
   * Called for every request. Add data using methods on the supplied {@link RequestTemplate}.
   */
  void apply(RequestTemplate template);
}
```

##### 实现自定义拦截器认证逻辑

```java
public class FeignAuthRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        // 模拟将 token 放到请求头进行传递，实际业务中可能传递就是用户Id等
        String access_token = UUID.randomUUID().toString();
        template.header("Authorization",access_token);
    }
}
```

##### 配置拦截器

```java
// Java代码全局配置
@Configuration
public class FeignConfig {
  
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }
  
    /**
     * 配置自定义拦截器
     */
    @Bean
    public FeignAuthRequestInterceptor feignAuthRequestInterceptor(){
        return new FeignAuthRequestInterceptor();
    }
  
}
```

```yaml
# yml局部配置
feign:
  client:
    config:
      mall-order:  #对应微服务
        requestInterceptors[0]:  #配置拦截器
          com.tuling.mall.feigndemo.interceptor.FeignAuthRequestInterceptor
```

##### 测试案例

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/7CB5ED24B4B14792A08D5F9642F2B37B/13364)

##### 服务端的处理

- 如上，mall-order 端可以通过 @RequestHeader 获取请求参数
- 一般在 Filter、Interceptor 中进行处理

### 超时时间配置

- 通过 Options 可以配置超时时间
  - 第一个参数：连接的超时时间，单位毫秒，默认2秒
  - 第二个参数：请求处理的超时时间，单位毫秒，默认5秒
- Feign 底层用的是 Ribbon，但超时时间以 Feign 配置为准

#### 代码配置

```java
// 全局配置
@Configuration
public class FeignConfig {
    @Bean
    public Request.Options options() {
        return new Request.Options(5000, 10000);
    }
}
```

#### yml配置

```yaml
feign:
  client:
    config:
      mall-order:  #对应微服务
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
```

#### 测试案例

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/E9434182210C457981177E16810016EA/13423)

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/0B0B92BB8D554A5BA50EBB64A1F8FD7C/13425)

### 客户端组件配置

- Feign默认使用 JDK 原生 URLConnection 发送 Http 请求
- 可集成组件进行替换，比如：
  - HttpClient
  - OkHttp
  - ...

#### Http调用底层执行逻辑

- **Feign.Client#execute**

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/3F5021F0C440477FA9087A51604629DF/13450)

#### 配置 HttpClient

##### 依赖

```xml
<!-- Apache HttpClient -->
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.7</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>10.1.0</version>
</dependency>
```

##### 配置

```yaml
feign:
  # feign 使用 Apache HttpClient  可以忽略，默认开启
  httpclient:
    enabled: true  
```

- 配置源码类：FeignAutoConfiguration
- 测试最终进入的方法为 feign.httpclient.ApacheHttpClient#execute

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/11035FB47EBE4334B076BF983EE6C28C/13462)

#### 配置 OkHttp

##### 依赖

```xml
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
</dependency>
```

##### 配置

```yaml
feign:
  #feign 使用 okhttp
  httpclient:
    enabled: false
  okhttp:
    enabled: true
```

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/B129568BA00B44A69F68567E3D9D3630/13439)

- 测试最终进入的方法为 feign.okhttp.OkHttp#execute

### GZIP压缩配置

- 开启压缩可以有效节约网络带宽，提升接口性能

##### 配置

```yaml
feign:
  # 配置 GZIP 来压缩数据
  compression:
    request:
      enabled: true
      # 配置压缩的类型
      mime-types: text/xml,application/xml,application/json
      # 最小压缩值
      min-request-size: 2048
    response:
      enabled: true
```

##### 注意

- Feign 客户端为 okHttp3 时，压缩配置不会生效

![](https://note.youdao.com/yws/public/resource/e4d3a42acab8240647293dde5ed88b7b/xmlnote/06A8CD5A91E74DB7B47F7FB3A0FC5AC2/13482)

### 编码解码配置

- Feign 提供自定义编码解码设置，也提供了多种实现，如：
  - Gson
  - Jaxb
  - Jackson
  - ...

#### 扩展点

```java
public interface Encoder {
    void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;
}

public interface Decoder {
    Object decode(Response response, Type type) throws IOException, DecodeException, FeignException;
}
```

#### 代码配置

```java
@Bean
public Decoder decoder() {
    return new JacksonDecoder();
}

@Bean
public Encoder encoder() {
    return new JacksonEncoder();
}
```

#### yml配置

```yaml
feign:
  client:
    config:
      mall-order:  #对应微服务
        # 配置编解码器
        encoder: feign.jackson.JacksonEncoder
        decoder: feign.jackson.JacksonDecoder
```

