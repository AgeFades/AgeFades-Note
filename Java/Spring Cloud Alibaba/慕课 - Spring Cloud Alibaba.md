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

