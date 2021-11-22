[TOC]

# 极客时间 - Prometheus + Grafana

## 监控模式分类

- `Metrics`
- `Logging`
- `Tracing`
- `Healthchecks`

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603272160345.png)

### Logging

- 通过日志监控，比如 操作系统日志、Java项目日志、各种服务日志...
- 日志一般分为两种:
  - 结构化日志 Structured
    - 例如: { "code": 200, msg: "success", ... }
  - 非结构化日志 Unstructured
    -  例如: [WARN] DB error: 500

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603272361083.png)

### Tracing

- 调用链模式监控
- 比如:
  - 客户端发起请求
  - -> 请求到达Nginx
  - -> 请求到达网关
  - -> 请求到达具体微服务
  - -> 服务处理请求
  - -> 响应网关、响应客户端

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603272538462.png)

### Metrics

- 通过 不同时间的离散数据点 进行监控
- 跟 Logging 方式比较相似
  - 不同点:
    - Logging 是文本的形式
    - Metrics 是 离散数值 的形式，是可以聚合的

### Healthchecks

- 健康检查，通过 心跳机制 确认服务是否存活
- 比如: Eureka 就是通过 Healthchecks 检查注册的微服务健康状态。

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603272699197.png)

### Prometheus 所属分类

- Prometheus 包含 Metrics 和 Healthchecks

### 分类的比较

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603272845514.png)

- CapEx: 搭建监控性能的研发成本
- OpEx: 运维成本
- Reaction: 监控性能、告警能力
- Inverstigation: 出现问题后，帮助排查问题的能力

### 分类的适用场景

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603273074296.png)

### Metrics 监控分层

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603273163731.png)

### Metrics 监控架构模式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603273247962.png) 

### Prometheus 监控 Metrics

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603273359534.png)

## Prometheus 简介

[Prometheus 官网](https://prometheus.io)

- Prometheus 是一款 开源监控工具
- 实质上是一个 时间序列数据库（时序数据库）TSDB
- golang 语言实现
- Soundcloud 研发，源于谷歌 borgmon
- 多维度（标签），拉模式（Pull-based）
- 支持 白盒 & 黑盒 监控，DevOps 友好
- 主要专注点是: Metrics & Altert (监控和告警)，而非 Logging / Tracing
- 社区生态丰富（多语言，各种 exporters）
- 单机性能: 每秒消费百万级时间序列，上千个 targets

## 时间序列（Time Series）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603273985183.png)

- 数据类型: 时间点 + 数据点
- 多个 时间点 + 数据点 组合，就形成了一个序列。
- 以 时间为横坐标，以 序列为纵坐标，就形成了 时间序列的矩阵。

### 时序数据库的流行趋势

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603274319387.png)

## Prometheus 架构设计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603274472523.png)

### Prometheus 2.0存储设计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603274916741.png)

## Prometheus 基本概念

### Metrics 种类

#### 计数器（Counter）

```shell
# 始终增加

# 比如: http请求数、下单数...
```

#### 测量仪（Gauge）

```shell
# 当前值的一次快照（snapshot）测量，可增可减

# 比如: 当前磁盘使用率、当前同时在线用户数...
```

#### 直方图（Histogram）

```shell
# 通过分桶（bucket）方式统计样本分布

# 比如: 请求延迟的分布，0-10ms的是多少，10ms-20ms的是多少...
```

#### 汇总（Summary）

```shell
# 根据样本统计出 百分位, 客户端计算

# 比如: 50% 是男客户，50% 是女客户
```

### Prometheus 和目标的交互方式 

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603275560247.png)

```shell
# Target 即 Prometheus 监控目标
	# 比如: 操作系统、Java 项目、数据库...
	
# Prometheus 要求 target 都暴露 metrics 以供 prometheus 抓取数据，
	# 默认 15s pull 一次
```

### Metrics 的具体形式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603275695867.png)

```shell
# 如上图，Prometheus 想要监控目标服务的 http 请求总数
```

### Metric 的采取形式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603275843082.png)

### Exporters

```shell
# OS(操作系统) - Node Exporter
	# 比如: Linux、Windows、Mac
	
# Database(数据库)
	# 比如: MySQL、Postgres、CouchDB...
	
# Messaging(消息中间件)
	# 比如: Kafka、RabbitMQ、NATS...
	
# Logging(日志系统)
	# 比如: ElasticSearch、Fluentd、Telegraf...
	
# Key-Value(键值对服务)
	# 比如: Redis、Memcached...
	
# WebServer(网站服务)
	# 比如: Apache、Nginx...

# Proxy(代理）
	# 比如: Haproxy、Varnish...
	
# DNS
	# 比如: BIND、PowerDNS、Unbound...
	
# BlackBox(黑盒)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603276173135.png)

### PromQL

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603276342751.png)

```shell
# Prometheus 通过埋点 metrics 端点，pull 数据存储到本地时序数据库后，
	# 提供了 PromQL 支持查询、聚合数据
```

#### 案例

```shell
# 求 http 请求异常百分比
sum(rate(http_requests_total{status="500"}[5m])) by (path) 
/
sum(rate(http_requests_total[5m])) by (path)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603276551098.png)

### 告警

```shell
# Prometheus 支持告警，在 Prometheus 服务上，使用 告警表达式 完成。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603332836417.png)

#### 案例

```shell
# 4小时内磁盘是否会满？
ALERT DiskWillFullIn4Hours IF predict_linear(node_filesystem_free[1h], 4 * 3600) < 0
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603332970253.png)

#### 告警组件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603333113233.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603333150686.png)

## Prometheus 起步查询实验

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603333872982.png)

```shell
# 通过一个 Http 服务器模拟器，暴露 metrics 端点，
	# Prometheus 定期 pull 埋点数据，
	
	# 获取 http 请求总数，http 请求延迟分布，
	
	# 通过 PromQL 展示到 Web UI 上。
```

### SpringBoot2.x + Prometheus

```shell
# SpringBoot2.x 集成 Prometheus 与 SpringBoot1.x 集成 Prometheus 差异较大。

# SpringBoot1.x 引入 simpleclient_spring_boot 官方提供jar

# SpringBoot2.x 不能再引入 simpleclient_spring_boot 官方jar，
	# 启动会报错，很多类找不到。
	
# SpringBoot2.x 集成 Prometheus 主要依赖两个 jar:
	# spring-boot-starter-actuator
	
	# micrometer-registry-prometheus
```

#### pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.0.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.agefades.demo</groupId>
    <artifactId>prometheus</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>prometheus</name>
    <description>springboot + prometheus</description>

    <properties>
        <java.version>11</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.12</version>
            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-json</artifactId>
            <version>5.3.7</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
            <version>1.5.5</version>
        </dependency>
      
      	<dependency>
            <groupId>io.github.mweirauch</groupId>
            <artifactId>micrometer-jvm-extras</artifactId>
            <version>0.2.0</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

#### yml

```yaml
server:
  port: 8081

spring:
  application:
    name: java

management:
  endpoint:
    metrics:
      enabled: true
  endpoints:
    web:
      base-path: /metrics
      exposure:
        include:
          - "*"
```

#### 核心类

```java
package com.agefades.demo.prometheus;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Metrics;
import lombok.extern.slf4j.Slf4j;

import java.util.Random;
import java.util.concurrent.TimeUnit;

/**
 * Http模拟器激活 线程
 *
 * @author DuChao
 * @date 2020/10/22 11:22 上午
 */
@Slf4j
public class ActivitySimulator implements Runnable {

    /**
     * 随机对象
     */
    private final Random rand = new Random();

    /**
     * 是否结束，默认否
     */
    private volatile boolean shutdown = false;

    /**
     * 用于计数 Http 请求总数的 Counter 类型 Metrics
     */
    private final Counter httpRequestTotal =
            Counter.builder("HTTP请求统计")
                    .description("HTTP请求统计")
                    .register(Metrics.globalRegistry);

    /**
     * Http 模拟器初始化
     */
    public void init() {
        int requestRate = 10;
        requestRate *= (5 + this.rand.nextInt(10));

        int nbRequests = this.giveWithUncertainty(requestRate, 5);
        for (int i = 0; i < nbRequests; i++) {
            this.httpRequestTotal.increment();
        }
    }

    /**
     * 结束
     */
    public void shutdown() {
        this.shutdown = true;
    }

    public int giveWithUncertainty(int n, int u) {
        int delta = this.rand.nextInt(n * u / 50) - (n * u / 100);
        return n + delta;
    }

    @Override
    public void run() {
        log.info("Simulator start...");
        while (!shutdown) {
            this.init();
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
    }

}
```

```java
package com.agefades.demo.prometheus;

import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.core.task.SimpleAsyncTaskExecutor;
import org.springframework.core.task.TaskExecutor;
import org.springframework.web.bind.annotation.RestController;

/**
 * http 模拟器启动类
 *
 * @author DuChao
 * @date 2020/10/22 11:56 上午
 */
@Slf4j
@RestController
@SpringBootApplication
public class PrometheusApplication implements ApplicationListener<ContextClosedEvent> {

    public static void main(String[] args) {
        SpringApplication.run(PrometheusApplication.class, args);
    }

    private ActivitySimulator simulator;

    @Bean
    MeterRegistryCustomizer<MeterRegistry> registryCustomizer() {
        return (registry -> registry.config().commonTags("application", "Swordsman Prometheus"));
    }

    @Bean
    public TaskExecutor executor() {
        return new SimpleAsyncTaskExecutor();
    }

    @Bean
    public CommandLineRunner runner(TaskExecutor executor) {
        return (args) -> {
            simulator = new ActivitySimulator();
            executor.execute(simulator);
            log.info("模拟器线程启动...");
        };
    }

    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        simulator.shutdown();
        log.info("Http模拟器关闭...");
    }

}
```

#### 说明

```shell
# 查看采集数据 metrics
http://localhost:8080/actuator/prometheus
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603354465043.png)

```shell
# 因为 prometheus 官方 jar 不支持 SpringBoot2.x，
	# 很多内容都不兼容了，所以 定制化埋点，这里本人没怎么研究明白，不再记录。
```

## Prometheus + Grafana 安装

```yaml
# prometheus.yml
global:
  scrape_interval:     15s # 抓取数据的间隔时间
  evaluation_interval: 15s # 抓取数据的超时时间

scrape_configs:
  - job_name: 'prometheus' # 抓取任务名称
    static_configs:
      - targets: ['localhost:9090']	# 目标服务 ip+port，这里是 prometheus 本身

  - job_name: 'vosp-admin' # SpringBoot服务
    metrics_path: '/metrics/prometheus' # 采集数据暴露端点
    static_configs:
      - targets: ['10.10.10.149:8081'] # ip + port，docker 启的不能监听到本地localhost
```

```shell
# docker run prometheus
docker run -d \
--name prometheus \
-p 9090:9090 \
-v /Users/apple/Documents/Docker/prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml \
prom/prometheus


# docker run grafana
docker run -d --name grafana -p 3000:3000 grafana/grafana
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603423307122.png)

```shell
# 访问 localhost:9090/targets，查看 prometheus 与 被收集数据服务 连通
```

```shell
# 访问 localhost:3000 ，账号密码默认都是 admin
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1621932993323.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603423434361.png)

```shell
# 开始导入 dashboard 仪表盘模板
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603423479717.png)

```shell
# https://grafana.com/grafana/dashboards/12856
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603423540712.png)

```shell
# 导入成功，设置基础信息，展示数据源 Prometheus
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603423586328.png)

```shell
# 完事
```

## 相关链接

[Grafana模板库](https://grafana.com/grafana/dashboards)

[Prometheus官网](https://prometheus.io/)