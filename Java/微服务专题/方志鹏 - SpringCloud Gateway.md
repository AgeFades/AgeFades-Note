[TOC]

# 方志鹏 - SpringCloud Gateway

## 官网

​	[https://spring.io/guides/gs/gateway](https://spring.io/guides/gs/gateway)

## 简介

​	Cloud 官方推出的第二代网关框架，取代 Zuul 网关。

### 	常见功能

​		路由转发

​		权限校验

​		限流控制

## 创建工程

```xml
   <!-- pom 依赖版本 -->
	<parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.5.RELEASE</version>	
    </parent>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Finchley.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
 	<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
	</dependency>
```

## 简单路由

​	Gateway 中使用 RouteLocator 的 Bean 进行路由转发

​	请请求进行处理，最后转发到目标的下游服务。

```java
@SpringBootApplication
@RestController
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    
    /**
     * RouterLocatorBuilder : 路由构建对象
     * 		routes() : 创建路由
     * 		route() : 制定路由规则
     *		path()	: 路由请求
     *		filters() : 路由中添加过滤器
     * 		uri()	: 路由导向路径
     * 总 : 让所有 /get 请求转发到 localhost:80/get
     *		中间 filter 会在请求头中添加 <k,v> 为 <Hello,World>
     */
    @Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
       return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://localhost"))
        .build();
    }
    
}
```

## 使用 Hystrix

​	在Gateway 中可以使用 Hystrix 做服务熔断降级。

​	在Gateway 中 Hystrix 是以  filter 的形式使用。

```java
   /**
   	* Gateway 中使用大量断言函数
    * host() : 断言请求 Host 中是否含该域名，进入路由规则
    * filters() : 此次加入 Hystrix 过滤器
    *	可配置名称、指向性 fallback 逻辑的地址
    */
	@Bean
    public RouteLocator myRoutes(RouteLocatorBuilder builder) {
        String httpUri = "http://httpbin.org:80";
        return builder.routes()
            .route(p -> p
                .path("/get")
                .filters(f -> f.addRequestHeader("Hello", "World"))
                .uri(httpUri))
            .route(p -> p
                .host("*.hystrix.com")
                .filters(f -> f
                    .hystrix(config -> config
                        .setName("mycmd")
                        .setFallbackUri("forward:/fallback")))
                .uri(httpUri))
            .build();
    }

	/**
	 * 熔断后处理逻辑
	 * Mono 是一个 Reactive stream ，对外输出的一个字符串。
	 * 具体熔断逻辑可自行操作
	 */
    @RequestMapping("/fallback")
        public Mono<String> fallback() {
            return Mono.just("fallback");
        }

	/**
	 *  curl --dump-header - --header 'Host:
	 *		www.hystrix.com'http://localhost:8080/delay/1
	 *	此时响应结果为 fallback
	 */
```

## Gateway 之 Predict 篇

### 	工作流程

#### 		常见作用

​			协议转换、路由转发

​			流量聚合，对流量进行监控，日志输出

​			作为整个系统的前端工程，可以限流

​			外部流量只能通过网关访问系统

​			网关权限校验

​			网关缓存

#### 		流程概览

​			客户端向 Gateway 发出请求，如果 Gateway Handler Mapping 确定请求与路由匹配（断言函数）

​			则将请求发送给 Web Handler 处理，在请求过程中经过一系列的过滤器链。

​			pre -> 代理请求 -> 响应 -> post

​			pre 中往往进行鉴权、限流、日志输出、请求头的更改、协议的转换 ...

​			post 中可以对响应体数据、协议转换 ...

​			重点 : 

​				请求与路由的匹配 : predicate					

​			![UTOOLS1561691166942.png](https://i.loli.net/2019/06/28/5d15841fd79d618590.png)	

#### 		predicate

​			JDK8接口，接收一个输入参数，返回一个 boolean

​			该接口包含多种默认方法来将 predicate 组成其他复杂逻辑（例 : 与、或、非）

​			可用于接口请求参数校验、判断新老数据是否有变化需要进行更新操作

​			add : 与  |  or : 或  |  negate : 非

​			Gateway 内置许多 predicate，源码在 org.springframework.cloud.gateway.handler.predicate​			![UTOOLS1561691821502.png](https://i.loli.net/2019/06/28/5d1586b47cda956254.png)

### 	predicate 实战

#### 		After Route Predicate Factory

```yaml
server:
  port: 8081 # 端口号
spring:
  profiles:
    active: after_route	# 指定程序的启动配置文件

---	# 在 application.yml 再建一个配置文件，语法是 --- 
spring:
  cloud:
    gateway:
      routes:
      - id: after_route	# 每个 route 的唯一id
        uri: http://httpbin.org:80/get # 将请求路由导向地址
        predicates:	# 请求匹配路由规则
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
        # 会被解析成 PredicateDefinition 对象(name = After,args = 2017...)
        # 遵循契约大于配置，实际被 AfterRoutePredicateFactory 所处理
        # 此 After 指定 Gateway web handler 中类为 AfterRoutePredicateFactory
        # 同理，其他 predicate 也遵循这个规则
  profiles: after_route	# 指定配置文件名
```

#### 		Header Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: header_route
        uri: http://httpbin.org:80/get
        predicates:
        - Header=X-Request-Id, \d+
        # Header 需要两个参数，一个是 header 名，一个是 header 值(可以是正则)
        # 以上为 : 请求 Header 中有 X-Request 的 Key，且 Value 为数字
        # curl -H 'X-Request-Id:1' localhost:8081
```

#### 		Cookie Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: cookie_route
        uri: http://httpbin.org:80/get
        predicates:
        - Cookie=name, forezp
		# 基本同 Header 断言一致，不同就是 Cookie 和 Header
		# curl -H 'Cookie:name=forezp' localhost:8081
```

#### 		Host Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: host_route
        uri: http://httpbin.org:80/get
        predicates:
        - Host=**.fangzhipeng.com
        # Host 一个参数，即 hostname
        # 支持 * 通配符
        # curl -H 'Host:www.fangzhipeng.com' localhost:8081
```

#### 		Method Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://httpbin.org:80/get
        predicates:
        - Method=GET
		# Method 一个参数 即 请求类型
		# curl localhost:8081  true
		# curl -XPOST localhost:8081  false
```

#### 		Path Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: path_route
        uri: http://httpbin.org:80/get
        predicates:
        - Path=/foo/{segment}
        # Path 一个参数 即 一个 spel 表达式，应用匹配路径
        # curl localhost:8081/foo/dew
```

#### 		Query Route Predicate Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: query_route
        uri: http://httpbin.org:80/get
        predicates:
        - Query=foo, ba.
		# Query 两个参数 一个参数名 一个参数值的正则
		# curl localhost:8081?foo=bar
		# 可以只有一个参数 即参数名 curl localhost:8081?foo=xxx
```

## Gateway 之 filter篇

### 	filter 的作用和生命周期

#### 		作用

​			多个服务需要做相同的事情，比如鉴权、限流、日志输出等。

​			Gateway 在微服务的上层搭建全局权限控制、限流、日志输出，再转发请求

#### 		生命周期

​		![UTOOLS1561698274202.png](https://i.loli.net/2019/06/28/5d159fe38251030118.png)	

#### 		gateway filter

​			针对单个路由规则

​			过滤器允许以某种方式修改 Http 请求 or Http 响应。

​			Gateway 内置许多 GatewayFilter 工程。

​			类似 predicate ，在 application.yml 中配置，遵循约定大于配置思想。

​			只需配置 GatewayFilter Factory 的名称，不需要全类名

​			例 : AddRequestHeaderGatewayFilterFactory 只需 AddRequestHeader

![UTOOLS1561698720938.png](https://i.loli.net/2019/06/28/5d15a1a27aa9a81195.png)

##### 			AddRequestHeader GatewayFilter Factory​				

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: add_request_header_route
        uri: http://httpbin.org:80/get
        filters:
        - AddRequestHeader=X-Request-Foo, Bar
        # 在请求头中添加一对 <K,V> <X-Request-Foo,Bar>
        # 类似 AddResponseHeader
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

```java
/**
 * 源码
 */
public class AddRequestHeaderGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory {

	@Override
	public GatewayFilter apply(NameValueConfig config) {
		return (exchange, chain) -> {
			ServerHttpRequest request = exchange.getRequest().mutate()
					.header(config.getName(), config.getValue())
					.build();

			return chain.filter(exchange.mutate().request(request).build());
		};
    }

}
```

##### 			RewritePath GatewayFilter Factory

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: rewritepath_route
        uri: https://localhost:9002
        predicates:
        - Path=/foo/**
        filters:
        - RewritePath=/foo/(?<segment>.*), /$\{segment}
        # 类似 Nginx 重写路径
        # 例 : localhost:9001/foo/add -> localhost:9002/add
```

#### 		自定义过滤器

```java
/**
 * 自定义过滤器需要实现 GatewayFilter + Ordered 两个接口
 */
public class RequestTimeFilter implements GatewayFilter, Ordered {

    private static final Log log = LogFactory.getLog(GatewayFilter.class);
    private static final String REQUEST_TIME_BEGIN = "requestTimeBegin";

    /**
     * 具体 Filter 逻辑
     */
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		
        // 存入 <K,V> 记录时间 pre Filter
        exchange.getAttributes().put(REQUEST_TIME_BEGIN, System.currentTimeMillis());
        // 记录请求耗时 post Filter
        return chain.filter(exchange).then(
                Mono.fromRunnable(() -> {
                    Long startTime = exchange.getAttribute(REQUEST_TIME_BEGIN);
                    if (startTime != null) {
                        log.info(exchange.getRequest().getURI().getRawPath() + ": " + (System.currentTimeMillis() - startTime) + "ms");
                    }
                })
        );

    }
	
    /**
     * Filter 的优先级，值越小优先级越高
     */
    @Override
    public int getOrder() {
        return 0;
    }
}
```

```java
	/**
	 * 将自定义过滤器注册到 router 中
	 */
	@Bean
    public RouteLocator customerRouteLocator(RouteLocatorBuilder builder) {
        // @formatter:off
        return builder.routes()
                .route(r -> r.path("*")
                        .filters(f -> f.filter(new RequestTimeFilter())    
                        .uri("http://httpbin.org:80/get")
                        .order(0)
                        .id("request_time_filter")
                )
                .build();
        // @formatter:on
    }
```

#### 		自定义过滤器工厂

​			为了在配置文件中配置自定义过滤器。

​			GatewayFilterfactory 层级

![image-20190628133233299](/Users/admin/Library/Application Support/typora-user-images/image-20190628133233299.png)

​			过滤器工厂的顶级接口是 GatewayFilterFactory，一般继承以下两个类进行开发

​			AbstractGatewayFilterFactory

​				接收一个参数

​				例 : RedirectToGatewayFilterFactory

​			AbstractNameValueGatewayFilterFactory

​				接收两个参数

​				例 : AddRequestHeaderGatewayFilterFactory

```java
public class RequestTimeGatewayFilterFactory extends AbstractGatewayFilterFactory<RequestTimeGatewayFilterFactory.Config> {


    private static final Log log = LogFactory.getLog(GatewayFilter.class);
    private static final String REQUEST_TIME_BEGIN = "requestTimeBegin";
    private static final String KEY = "withParams";

    /**
     * 一定要重写的方法
     */
    @Override
    public List<String> shortcutFieldOrder() {
        return Arrays.asList(KEY);
    }

    /**
     * 一定要调用父类构造器，将 Config 传过去
     * 否则会报 ClassCastException
     */
    public RequestTimeGatewayFilterFactory() {
        super(Config.class);
    }
	
    /**
     * 创建一个 GatewayFilter 的匿名内部类
     */
    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            exchange.getAttributes().put(REQUEST_TIME_BEGIN, System.currentTimeMillis());
            return chain.filter(exchange).then(
                    Mono.fromRunnable(() -> {
                        Long startTime = exchange.getAttribute(REQUEST_TIME_BEGIN);
                        if (startTime != null) {
                            StringBuilder sb = new StringBuilder(exchange.getRequest().getURI().getRawPath())
                                    .append(": ")
                                    .append(System.currentTimeMillis() - startTime)
                                    .append("ms");
                            // 是否打印请求参数的开关
                            if (config.isWithParams()) {
                                sb.append(" params:").append(exchange.getRequest().getQueryParams());
                            }
                            log.info(sb.toString());
                        }
                    })
            );
        };
    }


    /**
     * 接收 boolean 参数
     */
    public static class Config {

        private boolean withParams;

        public boolean isWithParams() {
            return withParams;
        }

        public void setWithParams(boolean withParams) {
            this.withParams = withParams;
        }

    }
}
```

```java
    /**
     * 最后在启动 Application 中注册 Bean
     */
	@Bean
    public RequestTimeGatewayFilterFactory elapsedGatewayFilterFactory() {
        return new RequestTimeGatewayFilterFactory();
    }
```

```yaml
spring:
  cloud:
    gateway:
      routes:
      - id: elapse_route	
        uri: http://httpbin.org:80/get
        filters:
        - RequestTime=false
        # 配置自定义过滤器，打印请求时间和参数
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
```

#### 		global filter

​			GatewayFilter : 需要通过 spring.cloud.routes.filters 配置在具体路由下

​				只作用在当前路由上或通过 spring.cloud.default.filters 配置在全局，作用在所有路由上

​			GlobalFilter : 不需要在配置文件中配置，作用在所有路由上

​				最终通过 GatewayFilterAdapter 包装成 GatewayFilterChain 可识别的过滤器

![UTOOLS1561701200527.png](https://i.loli.net/2019/06/28/5d15ab52a4f5966301.png)

##### 			自定义全局过滤器	

```java
/**
 * 对请求参数中的 token 进行鉴权
 */
public class TokenFilter implements GlobalFilter, Ordered {

    Logger logger=LoggerFactory.getLogger( TokenFilter.class );
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null || token.isEmpty()) {
            logger.info( "token is empty..." );
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return -100;
    }
}
```

```java
/**
 * 启动类中注册
 */
@Bean
public TokenFilter tokenFilter(){
        return new TokenFilter();
}
```

## Gateway 之 限流篇

### 	简介

​		高并发系统中，限流一方面防止大量请求使服务器过载，导致服务不可用，另一外面防止网络攻击。

​		常见限流 : 

​			Hystrix 使用线程池（信号量）隔离、超过线程池负载，走熔断逻辑。

​			Tomcat 通过限制线程数控制并发。

​			也有通过时间窗口平均速度来控制流量。

​			Ip 限流，uri 限流，用户访问频次限流 ...

​		应用层级 :

​			一般都是在网关做，比如 Nginx、Openrestry、Kong、Zuul ...

​			也可以在应用层通过 Aop 做

### 	常见的限流算法

#### 		计数器算法

​			比如限流 qps 为100，算法的实现思路就是从第一个请求进来开始计时

​			在接下来的 1s 内，每来一次请求，就计数 + 1，达到 100 则拒绝后续请求

​			1s 后恢复成0 重新计数。

​			具体实现 : 

​				每次服务调用，通过 AtomicLong.incrementAndGet() +1 并返回最新值

​				通过最新值和 阙值比较。

​			弊端 : 如果 1s 前 10ms，已经达到 100个请求，后续 990ms 全都拒绝。

​				  称为 "突刺现象"

### 		漏桶算法

​			为了消除 "突刺现象"

​			水龙头不管多大水，烂桶子只有一个小洞。

​			请求变化多大，都不影响处理的恒定速率。

![UTOOLS1561702589489.png](https://i.loli.net/2019/06/28/5d15b0beb701894206.png)

​			具体实现 :

​				准备一个队列，保存请求。通过一个线程池（ScheduledExecutorService）

​				定期从队列中获取请求并执行，可一次性获取多个任务并发执行。

​			弊端 : 

​				无法应对短时间的突发流浪，桶子满了就溢出（请求多了就抛弃请求）

#### 		令牌桶算法

​			令牌通算法能够在限制调用的平均速率的同时，还允许一定程度的突发调用。

​			桶，存放固定数量的令牌。

​			机制，以一定的速率往桶中放令牌。

​			请求需拿到令牌才能执行，否则等待 or 直接拒绝

​			例 : 设置 qps 为100，限流器初始化完成 1s 后，桶中 100个令牌。

​				开始对外服务时，可以顺挡 100 个请求

![UTOOLS1561703261440.png](https://i.loli.net/2019/06/28/5d15b35ee80a061623.png)

​			实现思路 :

​				一个队列，保存令牌，另外一个通过线程池定期生成令牌放到队列

​				请求 -> 队列中拿到令牌 -> 线程执行

### 	Gateway 限流

​		限流作为网关最基本的功能，Gateway 提供了 RequestRateLimiterGatewayFilterFactory

​			使用 redis 和 lua 实现了令牌桶，具体实现逻辑自行参考源码

![UTOOLS1561703502107.png](https://i.loli.net/2019/06/28/5d15b44f200b123454.png)

### 	使用案例

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifatId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

```yaml
server:
  port: 8081
spring:
  cloud:
    gateway:
      routes:
      - id: limit_route
        uri: http://httpbin.org:80/get
        predicates:
        - After=2017-01-20T17:42:47.789-07:00[America/Denver]
        filters:
        - name: RequestRateLimiter
          args:
            key-resolver: '#{@hostAddrKeyResolver}' # 限流的键的解析器的 Bean 对象名
            										# SpEL表达式 #{@beanName}
            redis-rate-limiter.replenishRate: 1 # 令牌桶每秒填充平均速率
            redis-rate-limiter.burstCapacity: 3 # 令牌桶总容量
  application:
    name: gateway-limiter
  redis:
    host: localhost
    port: 6379
    database: 0
```

```java
/**
 * 自定义限流规则
 * Hostname 限流
 */
public class HostAddrKeyResolver implements KeyResolver {

   	/**
   	 * 限流规则
   	 */
    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.just
            (exchange.getRequest()
             		 .getRemoteAddress()
             		 .getAddress()
             		 .getHostAddress());
    }

}

 	@Bean
    public HostAddrKeyResolver hostAddrKeyResolver() {
        return new HostAddrKeyResolver();
    }

```

```java
/**
 * Uri 限流
 */
public class UriKeyResolver  implements KeyResolver {

    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        return Mono.just
            (exchange.getRequest()
             		 .getURI()
             		 .getPath());
    }

}

 	@Bean
    public UriKeyResolver uriKeyResolver() {
        return new UriKeyResolver();
    }
```

```java

	/**
	 * 用户维度限流
	 */
   @Bean
    KeyResolver userKeyResolver() {
        return exchange -> 
            Mono.just(exchange.getRequest()
                      		  .getQueryParams()
                      		  .getFirst("user"));
    }

```

​		可以使用 jmeter 测压，redis 存了两个 key

![UTOOLS1561705486591.png](https://i.loli.net/2019/06/28/5d15bc0ee52e829970.png)

## Gateway 之服务注册与发现

### 	工程介绍

![UTOOLS1561705833016.png](https://i.loli.net/2019/06/28/5d15bd6a888bf64196.png)

```yaml
server:
  port: 8081

spring:
  application:
    name: sc-gateway-service
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true # 开启服务注册与发现功能
          				# 自动根据服务发现为每一个服务创建 router
          				# 该 router 将以服务名开头的请求路径转发到对应服务
          lowerCaseServiceId: true # 将服务名配置为小写
          
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

```yaml
spring:
  application:
    name: sc-gateway-server
  cloud:
    gateway:
      discovery:
        locator:
          enabled: false # 关闭服务发现自动注册 router
          lowerCaseServiceId: true
      routes:	# 自定义 router 规则
      - id: user
        uri: lb://USER # user 服务的负载均衡地址
        predicates:
          - Path=/user/**	# 断言函数匹配路径
        filters:
          - StripPrefix=1	# 转发之前去掉前缀 /user/add -> /add
```

