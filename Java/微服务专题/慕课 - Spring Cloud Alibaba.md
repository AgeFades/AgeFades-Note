[TOC]

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

## Spring Cloud Alibaba

### 项目整合 Spring Cloud Alibaba

```xml
 <dependencyManagement>
        <dependencies>
            <!--整合spring cloud-->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR1</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <!--整合spring cloud alibaba-->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>2.1.0.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
        		</dependency>
        </dependencies>
    </dependencyManagement>
```

## Nacos

### 搭建 Nacos Server

[Nacos 下载地址]: https://github.com/alibaba/nacos/releases

```shell
# 本人采用 docker 安装
# 拉取镜像 <此时最新版本为 1.1.4>
docker pull nacos/nacos-server

# MySQL 中创建 nacos 数据库，并执行 sql 脚本

# 创建挂载目录
mkdir -p /docker/nacos/conf
mkdir -p /docker/nacos/data

# 创建配置文件
vim /docker/nacos/conf/application.properties

# 编写配置文件如下:
```

```properties
# spring

server.contextPath=/nacos
server.servlet.contextPath=/nacos
server.port=8848

# nacos.cmdb.dumpTaskInterval=3600
# nacos.cmdb.eventTaskInterval=10
# nacos.cmdb.labelTaskInterval=300
# nacos.cmdb.loadDataAtStart=false


# metrics for prometheus
#management.endpoints.web.exposure.include=*

# metrics for elastic search
management.metrics.export.elastic.enabled=false
#management.metrics.export.elastic.host=http://localhost:9200

# metrics for influx
management.metrics.export.influx.enabled=false
#management.metrics.export.influx.db=springboot
#management.metrics.export.influx.uri=http://localhost:8086
#management.metrics.export.influx.auto-create-db=true
#management.metrics.export.influx.consistency=one
#management.metrics.export.influx.compressed=true

server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D %{User-Agent}i
# default current work dir
server.tomcat.basedir=

## spring security config
### turn off security
#spring.security.enabled=false
#management.security=false
#security.basic.enabled=false
#nacos.security.ignore.urls=/**

nacos.security.ignore.urls=/,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/login,/v1/console/health/**,/v1/cs/**,/v1/ns/**,/v1/cmdb/**,/actuator/**,/v1/console/server/**

# nacos.naming.distro.taskDispatchPeriod=200
# nacos.naming.distro.batchSyncKeyCount=1000
# nacos.naming.distro.syncRetryDelay=5000
# nacos.naming.data.warmup=true
# nacos.naming.expireInstance=true

nacos.istio.mcp.server.enabled=false

# 表明用MySQL作为后端存储
spring.datasource.platform=mysql

# 有几个数据库实例
db.num=1

# 注意修改自己IP、PORT、USERNAME、PASSWORD
db.url.0=jdbc:mysql://localhost:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user=root
db.password=root
```

```shell
# 启动容器
docker run -p 8848:8848 \
--name nacos-server \
-e MODE=standalone \
--privileged=true \
-v /Users/apple/Documents/Docker/nacos/conf/application.properties:/conf/application.properties \
-d docker.io/nacos/nacos-server

# 访问 Web 端
localhost:8848/nacos/index.html

# 默认账号密码均为 nacos

# 集群搭建暂不演示，请参考:
http://www.imooc.com/article/288153
```

### 将应用注册到 Nacos

```xml
<!-- pom 增加依赖 -->
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
# yaml 增加配置
spring:
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
  application:
  	name: xxx	# 服务名
```

### 服务发现的领域模型

![UTOOLS1574659118721.png](https://i.loli.net/2019/11/25/mnGyOhXpFT86u93.png)

```shell
# NameSpace:
	# 实现隔离，默认 public
	
# Group:
	# 不同服务可以分到一个组，默认 DEFAULT_GROUP
	
# Service:
	# 微服务
	
# Cluster:
	# 对指定微服务的一个虚拟划分，默认 DEFAULT
	
# Instance:
	# 微服务实例
```

```yaml
# yaml 配置
cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 # nacos 注册地址
        namespace: # 命名空间
        cluster-name: # 集群名称
```

![UTOOLS1574659870357.png](https://i.loli.net/2019/11/25/B43VLQmzYMsDluN.png)

### Nacos 元数据

#### 元数据是什么？

```shell
# Metadata:
	# Nacos 数据<如配置和服务> 描述信息，如服务版本、权重、容灾策略、负载均衡策略、鉴权配置、各种自定义标签，从作用范围来看，分为服务级别的元信息、集群的元信息及实例的元信息。
```

#### 元数据作用

```shell
# 提供描述信息

# 让微服务调用更加灵活
```

#### 如何设置元数据

```shell
# 控制台设置<不推荐>

# 配置文件中设置
```

```yaml
cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 # nacos 注册地址
        metadata:
          user: swordsman	# 随便写，只要是 KV 形式就行
          password: swordsman
```

## Ribbon

### 如何实现负载均衡

```shell
# 负载均衡的两种方式
	# 服务器端负载均衡
	# 客户端负载均衡
```

### 手写一个客户端负载均衡器

```java
// 随机算法<原理就是随机在 List 中选取出一个元素>
ThreadLocalRandom.current().nextInt('对应服务的集群'.size());
```

### 使用 Ribbon 实现负载均衡

#### Ribbon 是什么

```shell
# Netflix 开源的客户端负载均衡器
```

#### Ribbon 使用

```java
// 在 restTemplate 上加注解
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
  return new RestTemplate();
}

// 示例
@GetMapping("/test")
public ApiResult test(){
  List<ServiceInstance> serviceInstances = discoveryClient.getInstances("user");
  log.info("用户中心服务列表: {}", serviceInstances);
  Object object = restTemplate.getForObject("http://user/sysUser/count", Object.class);
  log.info("返回结果: {}", object);
  return new ApiResult(Status.SUCCESS);
}
```

#### Ribbon 组成

| 接口                     | 作用                         | 默认值                                                       |
| ------------------------ | ---------------------------- | ------------------------------------------------------------ |
| IClientConfig            | 读取配置                     | DefaultClientConfigImpl                                      |
| IRule                    | 负载均衡规则，选择实例       | ZoneAvoidanceRule                                            |
| IPing                    | 筛选掉ping不通的实例         | DummyPing                                                    |
| ServerList<Server>       | 交给 Ribbon 的实例列表       | Ribbon: ConfigurationBasedServerList<br />Spring Cloud Alibaba: NacosServerList |
| ServerListFilter<Server> | 过滤掉不符合条件的实例       | ZonePreferenceServerListFilter                               |
| ILoadBalancer            | Ribbon 的入口                | ZoneAwareLoadBalancer                                        |
| ServerListUpdater        | 更新交给Ribbon 的List 的策略 | PollingServerListUpdater                                     |

#### Ribbon 内置的负载均衡规则

| 规则名称                    | 特点                                                         |
| --------------------------- | ------------------------------------------------------------ |
| AvailabilityFilteringRule   | 过滤掉一直连接失败的被标记为 circuit tripped 的后端 Server<br />并过滤掉那些高并发的后端 Server 或者使用一个 AvailabilityPredicate 来包含过滤 Sserver 的逻辑<br />其实就是检查 status 里记录的各个 Server 的运行状态 |
| BestAvailableRule           | 选择一个最小的并发请求的 Server，逐个考察 Server，如果 Server 被 tripped 了，则跳过 |
| RandomRule                  | 随机选择一个 Server                                          |
| ResponseTimeWeightedRule    | 已废弃，作用同 WeightedResponseTimeRule                      |
| RetryRule                   | 对选定的负载均衡策略机上重试机制，在一个配置时间段内当选择 Server 不成功，则一直尝试使用 subRule 的方式选择一个可用的 Server |
| RoundRobinRule              | 轮询选择，轮询 index，选择 index 对应位置的 Server           |
| WeightedResponseTimeRule    | 根据响应时间加权，响应时间越长，权重越小，被选中的可能性越低 |
| ZoneAvoidanceRule<默认规则> | 复合判断 Server 所在 Zone 的性能和 Server 的可用性选择 Server，在没有 Zone 的环境下，类似于轮询 |

#### 细粒度配置自定义

```java
// Java 代码配置 不推荐，可能产生 Spring 父子上下文的问题
```

```yaml
# 用配置属性配置
user:	# 为细粒度的具体服务配置 ribbon 轮询规则
	ribbon:
		NFLoadBalanceRuleClassName: com.netflix.loadbalancer.RandomRule # 配置随机轮询
```

#### 全局配置

```shell
@RibbonClients(defaultConfiguration=xxx.class)
```

#### 支持的配置项

```yaml
# 用配置属性配置
user: # 为细粒度的具体服务配置
	ribbon:
		NFLoadBalancerClassName: # ILoadBalancer 实现类
		NFLoadBalancerRuleClassName: # IRule 实现类
		NFLoadBalancerPingClassName: # IPing 实现类
	  NIWSServerListClassName: # ServerList 实现类
	  NIWSServerListFilterClassName: # ServerListFilter 实现类
```

#### 饥饿加载

```shell
# 默认情况下 Ribbon 是懒加载的，所以在第一次调用 restTemplate 的时候，响应是比较慢的。
```

```yaml
# 用配置属性配置
ribbon:
	eager-load:
		enabled: true # 开启饥饿加载
		clients: xxx # 为哪些服务开启饥饿加载，细粒度配置，多个用逗号分隔
```

#### 扩展Ribbon - 权重支持

```shell
# 在 Nacos 的控制台中，可以更改每个实例的权重

# 在 Ribbon 的八种负载均衡规则中，是没有支持权重的，所以需要自己扩展。
```

```java
@Slf4j
public class NacosWeightedRule extends AbstractLoadBalancerRule {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
        // 读取配置文件，并初始化 NacosWeightedRule
    }

    @Override
    public Server choose(Object key) {
        try {
            BaseLoadBalancer loadBalancer = (BaseLoadBalancer) this.getLoadBalancer();
            log.info("lb = {}", loadBalancer);

            // 想要请求的微服务的名称
            String name = loadBalancer.getName();

            // 拿到服务发现的相关 API
            NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();

            // Nacos Client 自动通过基于权重的负载均衡算法，选择一个实例
            Instance instance = namingService.selectOneHealthyInstance(name);

            log.info("port = {}, instance = {}", instance.getPort(), instance);

            return new NacosServer(instance);
        } catch (NacosException e) {
            return null;
        }
    }

}
```

#### 扩展Ribbon - 同集群优先

```java
@Slf4j
public class NacosSameClusterWeightedRule extends AbstractLoadBalancerRule {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) { }

    @Override
    public Server choose(Object key) {
        // 拿到配置文件中的集群名称
        String clusterName = nacosDiscoveryProperties.getClusterName();

        BaseLoadBalancer loadBalancer = (BaseLoadBalancer) this.getLoadBalancer();
        log.info("lb = {}", loadBalancer);

        // 想要请求的微服务的名称
        String name = loadBalancer.getName();

        // 拿到服务发现的相关 API
        NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();

        try {
            // 找到指定服务的所有实例 A
            List<Instance> instances = namingService.selectInstances(name, true);

            // 过滤出相同集群下的所有实例 B
            List<Instance> sameClusterInstances = instances.stream()
                    .filter(instance -> Objects.equals(instance.getClusterName(), clusterName))
                    .collect(Collectors.toList());

            List<Instance> instancesToBeChosen = new ArrayList<>();
            // 如果 B 是空，就用 A
            if (CollUtil.isEmpty(sameClusterInstances)) {
                instancesToBeChosen = instances;
                log.info("发生跨集群的调用，name = {}, clusterName = {}, instances = {}",name, clusterName, instances);
            } else {
                instancesToBeChosen = sameClusterInstances;
            }

            // 基于权重的负载均衡算法，返回1个实例
            Instance instance = ExtendBalancer.getHostByRandomWeight2(instancesToBeChosen);
            log.info("选择的实例是 port : {}, instance : {}", instance.getPort(),instance);
            return new NacosServer(instance);
        } catch (NacosException e) {
            log.error("ribbon获取服务失败,失败原因 : {}", e);
            return null;
        }
    }
}

class ExtendBalancer extends Balancer {
    public static Instance getHostByRandomWeight2(List<Instance> hosts) {
        return getHostByRandomWeight(hosts);
    }
}
```

### 深入理解 Nacos 的 Namespace

```shell
# Ribbon 只能调用相同 NameSpace 下的服务 

# Nacos 中利用 NameSpace 机制对服务做了隔离
```

### 现有架构存在的问题

```shell
# 代码不可读

# 复杂的 URL 难以维护

# 难以响应需求的变化

# 编程的体验不统一
```

## 使用Feign 实现远程 HTTP 调用

### 什么是 Feign

```shell
# Fegin 是Netflix 开源的声明式 HTTP 客户端

# Fegin 整合了 Ribbon 的负载均衡，所以上面的配置都是有用的
```

### 使用 Feign

```xml
<!-- 加依赖 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```shell
# 启动类上加注解
@EnabelFeignClients
```

```java
// 示例
@FeignClient(name = "user") // name 表示调用的服务名
public interface UserClient {

    /**
     * 统计系统用户数量
     */
    @GetMapping("/sysUser/page")
    ApiResult page(@RequestBody PageRequest pageRequest);
}
```

### Feign 的组成

| 接口               | 作用                                     | 默认值                                                       |
| ------------------ | ---------------------------------------- | ------------------------------------------------------------ |
| Feign.Builder      | Feign 的入口                             | Feign.Builder                                                |
| Client             | Feign 底层用什么去请求                   | 和 Ribbon 配合时: LoadBalanceFeignClient<br />不和 Ribbonn 配合时，Feign.Client.Default |
| Contract           | 契约，注解支持                           | SpringMvcContract                                            |
| Encoder            | 编码器，用于将对象转换成 HTTP 请求消息体 | SpringEncoder                                                |
| Decoder            | 解码器，将响应消息体转成对象             | ResponseEntityDecoder                                        |
| Logger             | 日志管理器                               | Slf4jLogger                                                  |
| RequestInterceptor | 用于为每个请求添加通用逻辑               | 无                                                           |

### 细粒度配置自定义

```shell
# Java 代码配置方式，不推荐，还是可能有Spring IOC 父子上下文重复扫描的问题

# 配置文件配置
```

#### 自定义 Feign 日志级别

| 级别                  | 打印内容                                         |
| --------------------- | ------------------------------------------------ |
| NONE<默认值>          | 不记录任何日志                                   |
| BASIC<适用于生产级别> | 仅记录请求方法、URL、响应状态代码以及执行时间    |
| HEADERS               | 记录 BASIC 级别的基础上，记录请求和响应的 header |
| FULL<适用于开发级别>  | 记录请求和响应的 header、body 和元数据           |

#### 配置属性方式指定 Feign 日志级别

```yaml
feign:
	client:
		conifg:
			# 想要调用的微服务的名称
			user:
				loggerLevel: full
```

### 全局配置自定义

```shell
# Java 代码方式:
	# 只建议在启动类上的 @EnabelFeignClients 注解上做配置

# 配置文件自定义
```

#### Java 代码配置

```java
// 自定义编写 GlobalFeignConfiguration 配置类，例如日志级别...
@EnabelFeignClients(defualtConfiguration = GlobalFeignConfiguration.class)
```

#### 配置属性方式

```yaml
feign:
	client:
		conifg:
			# 全局配置
			default:
				loggerLevel: full
```

### 支持的配置项

#### 支持的配置项 - 代码方式

| Logger.Level                   | 作用                                                   |
| ------------------------------ | ------------------------------------------------------ |
| Logger.Level                   | 指定日志级别                                           |
| Retryer                        | 指定重试策略                                           |
| ErrorDecoder                   | 指定错误解码器                                         |
| Request.Options                | 超时时间                                               |
| Collection<RequestInterceptor> | 拦截器                                                 |
| SetterFactory                  | 用于设置 Hystrix 的配置属性，Feign 整合 Hystrix 才会用 |

#### 支持的配置项 - 配置方式

```yaml
feign:
	client:
		config:
			# 服务名
			<feignName>:
				connectTimeout: 5000 # 连接超时时间
				readTimeout: 5000 # 读取超时时间
				loggerLevel: full # 日志级别
				errorDecoder: com.example.SimpleErrorDecoder # 错误解码器
				retryer: com.example.SimpleRetryer	# 重试策略
				requestInterceptors:
					- com.example.FooRequestInterceptor # 拦截器
				decode404: false # 是否对 404 错误码解码
				encoder: com.example.SimpleEncoder # 编码器
				decoder: com.example.SimpleDecoder # 编码器
				contract: com.example.SimpleContract # 契约
```

### 配置最佳实践总结

```yaml
# Feign 配置属性的优先级
全局代码配置 < 全局属性配置 < 细粒度代码配置 < 细粒度属性配置

# 尽量使用属性配置，尽量保持单一性
```

### Feign 的继承

```shell
# 官方不建议使用

# 很多公司使用
```

### 多参数请求构造

```yaml
# 加上配置，解决两个 FeignClient 代理重复的问题
spring:
	main:
		allow-bean-definition-overriding: true
```

```java
// 在 FeignClient 方法参数中加上注解，主要是针对 GET 请求，需要在 url 上拼装参数
@SpringQueryMap
```

### Feign 脱离 Ribbon 使用

```shell
# 目前阶段的 Feign 调用服务都是已经注册在 Nacos 上的服务，如果要调用没有注册到注册中心的服务，怎么办
```

```java
@FeignClient(name = "服务名", url = "服务地址")

// 举例
@FeignClient(name = "baidu", url = "http://www.baidu.com")
```

### RestTemplate VS Feign

| 角度             | RestTemplate | Feign                            |
| ---------------- | ------------ | -------------------------------- |
| 可读性、可维护性 | 一般         | 极佳                             |
| 开发体验         | 欠佳         | 极佳                             |
| 性能             | 很好         | 中等<RestTemplate 的 50%左右>    |
| 灵活性           | 极佳         | 中等<内置功能可满足绝大多数需求> |

### Feign 性能优化

```shell
# 为 Feign 配置连接池，性能提升 15% 左右

# Feign 支持 HttpClient 和 Ok Http，配置思路基本一致，okHttp 需要自己指定版本

# 日志级别配置为 BASIC 就可以
```

```xml
<!-- 加依赖 -->
<dependency>
		<groupId>io.github.openfeign</groupId>
  	<artifactId>feign-httpclient</artifactId>
</dependency>
```

```yaml
# 改配置
feign:
	httpclient:
		# 让 feign 使用 apache httpclient 做请求，而不是默认的 urlConnetction
		enabled: true
		max-connections: 200 # feign 的最大连接数
		max-connections-per-route: 50 # feign 单个路径的最大连接数
```

## Sentinel

### 雪崩效应

![UTOOLS1574832411109.png](https://i.loli.net/2019/11/27/xqjWyfdtgPVIKeU.png)

### 常见容错方案

```shell
# 请求超时释放线程

# 限流，控制阻塞线程最大数

# 仓壁模式

# 断路器模式
```

![UTOOLS1574833034965.png](https://i.loli.net/2019/11/27/Yz8x2GbMrkIC9Oi.png)

### 使用 Sentinel 实现容错

#### Sentinel 是什么

```shell
# 轻量级的流量控制、熔断降级 Java 库
```

#### 整合 Sentinel

```xml
<!-- 整合 Sentinel 的依赖，此时会暴露出/actuator/sentinel 的端点 -->
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

#### Sentinel 控制台

```shell
# 官方提供的 Sentinel 可视化界面
```

```shell
# 本人使用 Docker 安装，此时最新版本为 1.6.3

# 拉取镜像
docker pull bladex/sentinel-dashboard:1.6.3

# 运行容器
docker run -d \
--name sentinel \
-p 8858:8858 \
bladex/sentinel-dashboard:1.6.3

# 浏览器访问 localhost:8858

# 默认账户密码均为 sentinel
```

```yaml
# 将应用注册到 Sentinel 中
spring:
	cloud:
		sentinel:
			transport:
				dashboard: localhost:8858
```

```shell
# 此时启动应用，并有流量进入，才会加载到 Sentinel 的监控列表中
	# 所以，Sentinel 监控是懒加载模式
```

#### 流控规则

![UTOOLS1574842120288.png](https://i.loli.net/2019/11/27/GFZQzvkfgeCMxhq.png)

![UTOOLS1574842219660.png](https://i.loli.net/2019/11/27/UlvWnEYbD3VcNrh.png)

```shell
# 直接
	# 就最直接的
	# 针对来源 是对系统内所有微服务而言，default 表示所有
	
# 关联
	# 当关联的资源达到阈值，就限流自己<保护模式>
	# 适用于 Selecte 接口 QPS 过快，而限流 Update 接口
	# 关联资源 即关联的 Api
	
# 链路
	# 只记录指定链路上的流量
	# 例如 A、B 方法都调用 C 方法，此时在 C 方法上增加限流链路规则，对象为 A，即针对A 调用限流，其他的链路流量都不限制
	# 在 C 方法上需要加注解 @SentinelResource("") 限流资源名
 	# 入口资源 是针对细粒度的 Api 入口
```

![UTOOLS1574842317713.png](https://i.loli.net/2019/11/27/GqxghoTe9Bf3lZK.png)

#### 流控效果

```shell
# 快速失败
	# 直接抛出异常
	
# Warm Up
	# 根据 codeFactor(默认3,冷加载因子) 的值，从 阈值/codeFactor，经过预热时长，才到达设置的 QPS 阈值
	# 例如 单机阈值为 100，预热时长为 10秒，冷加载因子默认是3，所以系统会用 100/3 作为最初的阈值，在10 s 内才达到 100
	# 适用于秒杀服务
	
# 排队等待
	# 匀速排队，让请求以均匀的速度通过，阈值类型必须设成 QPS，否则无效
	# 例如: 单机阈值为 1，超时时间为 2s，在阻塞的前两秒内等待，超时后报出异常
	# 适用于突发流量
```

### 降级规则详解

![UTOOLS1575252443000.png](https://i.loli.net/2019/12/02/3tYiFr14NE8XIDh.png)

#### 降级策略详解

```shell
# RT
	# 平均响应时间
	# 平均响应时间(秒级统计) 超出阈值 && 在时间窗口内通过的请求 >= 5 ->
		# 触发降级(断路器打开) ->
		# 时间窗口结束 ->
		# 关闭降级
		
	# 注意点:
		# 默认 RT 最大 4900ms
		# 通过 -Dcsp.sentinel.statistic.max.rt=xxx 修改
		

# 异常比例
	# QPS >= 5 && 异常比例(秒级统计) 超过阈值 ->
		# 触发降级(断路器打开) ->
		# 时间窗口结束 -> 
		# 关闭降级

# 异常数
	# 异常数（分钟统计）超过阈值 ->
		# 触发降级(断路器打开) ->
		# 时间窗口结束 ->
		# 关闭降级
		
	# 注意点:
		# 时间窗口小于 60秒可能会出现问题
		
# Sentinel 的断路器目前没有半开状态
```

### 热点规则详解

![UTOOLS1575265339581.png](https://i.loli.net/2019/12/02/8x4jEIseOt2nMJG.png)

```shell
# Sentinel 控制台里默认是不支持热点规则的，需要自己写一些代码

# 对接口的请求参数某些值做限流规则，并且需要在端点上加 @SentinelResource 注解

# 注意点:
	# 参数必须是基本类型或者 String
```

### 系统规则详解

```shell
# LOAD
	# 当系统 load1(1分钟的 load) 超过阈值，且并发线程数超过系统容量时触发，建议设置为 CPU 核数 * 2.5
	# 仅对 Linux / Unix - like 机器生效
	
	# 系统容量 = maxQps * minRt
		# maxQps: 秒级统计出来的最大 QPS
		# minRt: 秒级统计出来的最小响应时间

# RT
	# 所有入口流量的平均 RT 达到阈值触发

# 线程数
	# 所有入口流量的并发线程数达到阈值触发

# 入口 QPS
	# 所有入口流量的 QPS 达到阈值触发
```

### 授权规则详解

![UTOOLS1575266005271.png](https://img03.sogoucdn.com/app/a/100520146/9f744a66e57894369d43123c90f5c220)

### Sentinel 与控制台通信原理剖析

#### 控制台是如何获取到微服务的监控信息的？

![UTOOLS1575266322733.png](https://img01.sogoucdn.com/app/a/100520146/94a9f66140c8b35ec000b87cffd621db)

#### 用控制台配置规则时，控制台是如何将规则发送到各个微服务的呢？

```shell
# 通过微服务注册在Sentinel 的地址通信，Sentinel 调用微服务的 setRules 端点。
```

### Sentinel API

```java
@GetMapping("/test-api")
public String testApi(@RequestParam(required = false) String a) {

	// 定义一个 Sentinel 保护的资源，名称是 test-api
  Entry entry = null;
  try {
    entry = SphU.entry("test-api");
    // 被保护的业务逻辑
    return a;
  } catch (BlockException e) {
    log.warn("限流或者降级了", e);
    return "限流或者降级了";
  } finally {
    if (entry != null) {
      // 退出 entry
      entry.exit();
    }
  }
  
}
```

### Feign 整合 Sentinel

```yaml
# 添加配置
feign:
	sentinel:
		enabled: true
```

#### 限流降级发生时，如何定制自己的处理逻辑？

```java
// 在 @FeignClient 注解上增加属性
@FeignClient(name = "user-center", fallback = Xxx.class)

// Xxx.class 需要实现 @FeignClient 标注的类，并加上 @Component 注解
```

#### 如何获取到异常

```java
// 在 @FeignClient 注解上增加属性，fallbackFactory 和 fallback 不能共存
@FeignClient(name = "user-center", fallbackFactory = Xxx.class)

// Xxx.class 需要实现 FallbackFactory<Xxx(@FeignClient 标注的类)>
```

### Sentinel 使用姿势总结

| 使用方式     | 使用方式                      | 使用方法                 |
| ------------ | ----------------------------- | ------------------------ |
| 编码方式     | API                           | try...catch...finally    |
| 注解方式     | SentinelResource              | blockHandler/fallback    |
| RestTemplate | SentinelRestTemplate          | blockHandler/fallback    |
| Feign        | feign.sentinel.enabled = true | fallback/fallbackFactory |

### 规则持久化

#### 拉模式

```shell

```

#### 推模式

```shell

```

## SpringCloud Gateway

### 为什么要使用网关

```shell
# 统一登录认证

# 对外只暴露一个域名

# 转换对浏览器不友好的协议通信
```

### SpringCloug Gateway 是什么？

```shell
# Spring Cloud 的第二代网关实现，未来会取代 Zuul

# 基于 Netty、Reactor以及 WebFlux 构建
```

### 编写 SpringCloud Gateway

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```yml
spring:
	cloud:
		gateway:
			discovery:
				locator:
					enbaled: true # 让 Gateway 通过服务发现组件找到其他的微服务
```

### 核心概念

#### Route

```shell
# 路由:
	# Gateway 的基础元素，可简单理解成一条转发的规则。
	# 包含: ID、目标Url、Predicate 集合以及 Filter 集合
```

#### Predicate

```shell
# 断言:
	# java.util.function.Predicate
	# Gateway 使用 Predicate 实现路由的匹配条件
```

#### Filter

```shell
# 过滤器:
	# 修改请求以及响应
```

#### 总结

```shell
# 路由 -> 转发规则

# 断言 -> 控制请求 -> 路由的条件

# 过滤器 -> 为路由添加业务逻辑
```

#### 路由配置示例

```yml
spring:
	cloud:
		gateway:
			routes:
				- id: some_route
					uri: http://www.baidu.com
					predicates: 
						- Path=/users/1
					filters:
						- AddRequestHeader=X-Request-Foo, Bar
```

```shell
# 一个路由规则由:
	# id、uri、断言集合、过滤器集合组成
	
# 上述意思是，当 /users/1 请求进入 Gateway 时，过滤器添加请求头信息，并转发到 uri 的位置
```

### SpringCloud Gateway 架构剖析

![UTOOLS1577340681325.png](https://i.loli.net/2019/12/26/IuVj1qZztQ6bHOf.png)

```shell
# Gateway Client:
	# 发起请求的客户端
	
# Proxied Service:
	# 网关代理的微服务
	
# Gateway HandlerMapping:
	# 判断请求是否符合路由的规则配置
		# 如果符合，进入 Gateway Web Handler 处理器 ->
		# 将请求交给过滤器集合进行逻辑处理
```

### 路由断言工厂详解

```shell
# Route Predicate Factories:
	# 符合 Predicate 的条件，就使用该路由的配置，否则就不管。
	
# Predicate 是 JDK8 提供的一个函数式编程接口
```

#### 路由配置的两种形式

##### 路由到指定 URL

```yaml
# 示例1: 通配
# 表示访问 GATEWAY_URL/** 会转发到 http://www.baidu.com/**
# 这段配置不能直接使用，需要和下面的 Predicate 配合使用才行
spring:
	cloud:
		gateway:
			routes:
				- id: {唯一标识}
					uri: http://www.baidu.com
```

```yaml
# 示例2: 精确匹配
# 表示访问 GATEWAY_URL/a/b 会转发到 http://www.baidu.com/a/b
# 这段配置不能直接使用，需要和下面的 Predicate 配合使用才行
spring:
	cloud:
		gateway:
			routes:
				- id: {唯一标识}
					uri: http://www.baidu.com/a/b
```

##### 路由到服务发现组件上的微服务

```yaml
# 示例1: 通配
# 表示访问 GATEWAY_URL/** 会转发到 user 微服务的 /**
# 这段配置不能直接使用，需要和下面的 Predicate 配合使用才行
spring:
	cloud:
		gateway:
			routes:
				- id: {唯一标识}
					uri: lb://user
```

```yaml
# 示例2: 精确匹配
# 表示访问 GATEWAY_URL/a/b 会转发到 user 微服务的 /a/b
# 这段配置不能直接使用，需要和下面的 Predicate 配合使用才行
spring:
	cloud:
		gateway:
			routes:
				- id: {唯一标识}
					uri: lb://user/a/b
```

#### 断言工厂详解

```shell
http://www.imooc.com/article/290804
```

### 过滤器工厂详解

```shell
http://www.imooc.com/article/290816
```

​	