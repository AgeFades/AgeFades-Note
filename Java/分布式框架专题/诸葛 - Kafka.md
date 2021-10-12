[TOC]

# 诸葛 - Kafka

[官方文档](https://kafka.apache.org/)

## 简介

- 一个 `分布式`、`支持分区(partition)`、`多副本(replica)` 基于 zookeeper 协调的 `分布式消息系统`
- 特性：实时处理大量数据，Scala编写
  - 基于 hadoop 的批处理系统
  - 低延迟的实时系统
  - Storm/Spark 流式处理引擎
  - Web/Nginx 日志
  - 访问日志
  - 消息服务
- 借鉴 JMS 规范思想，并没有完全遵循 JMS 规范

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/F90CBD18EAC24313A72BD593EFEE9F7D/82849)

## 基本概念

### Broker

- 一个 kafka 节点就是一个 broker
- 一个或多个 broker 组成一个 kafka 集群

### Topic

- producer 发往 broker 的每条 message 都要指定一个 topic
- kafka 根据 topic 对 message 做分组归类

### Producer

- message 生产者，向 broker 发送消息的客户端

### Consumer

- message 消费者，从 broker 读取消息的客户端

### ConsumerGroup

- 每个 consumer 属于一个特定的 consumer group
- 一条 message 可以被多个不同的 consumer group 消费
- 但是一个 consumer group 只能有一个 consumer 能够消费该 message

### Partition

- 物理概念，一个 topic 可以分为多个 partition
- 每个 partition 内部 message 是有序的

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/317C13052E514DA9B6229368DD48EDB5/105252)

### 小结

- producer 通过 tcp 发送 message 到 kafka 集群
- consumer 通过 tcp 拉取 message 到本地消费

## 安装部署

[参考链接](https://www.jianshu.com/p/b811ea29428c)

- 演示 docker-compose 一步安装 zk 集群、kafka 集群

```yaml
version: '3.1'
services:
  zoo1:
    image: zookeeper:3.4.13
    restart: always
    hostname: zoo1
    container_name: zoo1
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo2:
    image: zookeeper:3.4.13
    restart: always
    hostname: zoo2
    container_name: zoo2
    ports:
      - "2182:2181"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo3:
    image: zookeeper:3.4.13
    restart: always
    hostname: zoo3
    container_name: zoo3
    ports:
      - "2183:2181"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  kafka1:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka1
    container_name: kafka1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka1
      KAFKA_LISTENERS: PLAINTEXT://kafka1:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://宿主机ip:9092
      KAFKA_ADVERTISED_PORT: 9092
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka2:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka2
    container_name: kafka2
    ports:
      - "9093:9093"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka2
      KAFKA_LISTENERS: PLAINTEXT://kafka2:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://宿主机ip:9093
      KAFKA_ADVERTISED_PORT: 9093
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka3:
    image: wurstmeister/kafka
    restart: always
    hostname: kafka3
    container_name: kafka3
    ports:
      - "9094:9094"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: kafka3
      KAFKA_LISTENERS: PLAINTEXT://kafka3:9094
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://宿主机ip:9094
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3

  kafka-manager:
    image: sheepkiller/kafka-manager
    restart: always
    hostname: kafka-manager
    container_name: kafka-manager
    ports:
      - "9000:9000"
    environment:
      ZK_HOSTS: zoo1:2181,zoo2:2181,zoo3:2181
    depends_on:
      - zoo1
      - zoo2
      - zoo3
```

### 检验测试案例

```shell
# 进入容器
docker exec -it kafka1 bash
```

#### 创建Topic

```shell
# 创建 topic
# --partition 5 表示该 topic 有5个分区
# --replication-factor 3 表示该 topic 备份因子为 3
# 2.13是scala的版本、2.7.1是kafka的版本
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --create --topic agefades --partitions 5 --zookeeper 172.17.0.1:2181 --replication-factor 3
```

#### 查看Topic

```shell
# 查看 topic
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --list --zookeeper zoo1:2181
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633747913621.png)

#### 生产消息

```shell
# 生产消息
/opt/kafka_2.13-2.7.1/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic agefades
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633746787920.png)

#### 消费消息

```shell
# 进入集群另一台容器
docker exec -it kafka3 bash

# 监听 topic 消费消息
# 默认消费最新的消息，如果想消费之前的消息，需要加上 --from-beginning
/opt/kafka_2.13-2.7.1/bin/kafka-console-consumer.sh --bootstrap-server kafka2:9093 --topic agefades --from-beginning
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633746836555.png)

#### 消费多Topic

```shell
# 在节点 kafka1 创建topic、生产消息
docker exec -it kafka1 bash

# 创建test1 topic
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --create --topic test1 --partitions 5 --zookeeper 172.17.0.1:2181 --replication-factor 3

# 创建test2 topic
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --create --topic test2 --partitions 5 --zookeeper 172.17.0.1:2181 --replication-factor 3

# test1 生产消息
/opt/kafka_2.13-2.7.1/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic test1

# test2 生产消息
/opt/kafka_2.13-2.7.1/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic test2
```

```shell
# 在节点 kafka2 监听 test1、test2 两个topic进行消费
/opt/kafka_2.13-2.7.1/bin/kafka-console-consumer.sh --bootstrap-server kafka2:9093 --whitelist "test1|test2"  --from-beginning
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633759996232.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760005947.png)

#### 单播消费

- 一条 message 只能被一个 consumer 消费
  - 类似 queue 模式
  - 只需让所有 consumer 在同一个 consumerGroup 即可

```shell
# kafka2 和 kafka3 两个节点监听同一个 topic，在同一个 consumerGroup
# kafka2 和 kafka3 都执行如下命令
/opt/kafka_2.13-2.7.1/bin/kafka-console-consumer.sh --bootstrap-server kafka2:9093 --consumer-property group.id=testGroup --topic test1 --from-beginning
```

```shell
# kafka1 往 topic 里生产消息
/opt/kafka_2.13-2.7.1/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic test1

# 如下图所示，一个 message 只会被一个 consumer 消费
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760330825.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760341548.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760357242.png)

#### 多播消费

- 一条 message 被多个 consumer 消费
  - 类似 publish / subscribe
  - 实现多播，只需要保证这些 consumer 属于不同的 consumerGroup 即可

```shell
# kafka2
/opt/kafka_2.13-2.7.1/bin/kafka-console-consumer.sh --bootstrap-server kafka2:9093 --consumer-property group.id=testGroup1 --topic test2 --from-beginning

# kafka3
/opt/kafka_2.13-2.7.1/bin/kafka-console-consumer.sh --bootstrap-server kafka2:9093 --consumer-property group.id=testGroup2 --topic test2 --from-beginning

# kafka1
/opt/kafka_2.13-2.7.1/bin/kafka-console-producer.sh --broker-list kafka1:9092 --topic test2
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760559920.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760575055.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760589515.png)

#### 查看消费组名

```shell
/opt/kafka_2.13-2.7.1/bin/kafka-consumer-groups.sh --bootstrap-server kafka1:9092 --list
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760838490.png)

#### 查看消费组的消费偏移量

```shell
/opt/kafka_2.13-2.7.1/bin/kafka-consumer-groups.sh --bootstrap-server kafka1:9092 --describe --group testGroup1
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760955360.png)

- current-offset：当前消费组的已消费偏移量
- log-end-offset：topic 对应 partition message 的结束偏移量(HW)
- lag：当前消费组未消费的消息数

#### 查看topic情况

```shell
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --describe --zookeeper zoo1:2181 --topic test1
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633762211083.png)

- 第一行是所有分区的概要信息，之后的每一行表示每一个 partition 的信息
- Leader：负责给定 partition 的所有读写请求
- replicas：表示某个 partition 在几个 broker 上存在备份
- isr：replicas 的一个子集，只列出当前还存活着的、并已同步备份了该 partition 的节点

#### 删除Topic

```shell
# 删除topic
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh --delete --topic agefades --zookeeper zoo1:2181
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633748072594.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633760018354.png)

#### zk查看kafka数据

```shell
# 进入 zk 容器中的 zk 客户端
docker exec -it zoo1 bin/zkCli.sh

# 查看zk的根目录kafka相关节点
ls /		

# 查看kafka节点
ls /brokers/ids	
```

## server.properties核心配置

| Property                   | Default                 | Description                                                  |
| -------------------------- | ----------------------- | ------------------------------------------------------------ |
| broker.id                  | 0                       | kafka集群中broker节点唯一id，非负整数                        |
| log.dirs                   | /tmp/kafka-logs         | 数据路径，该路径不唯一，可以是多个，多个之间用逗号分隔。每次创建新 partition 时，都会选择在包含最少 partitions 的路径下进行 |
| listeners                  | PLAINTEXT://kafka1:9092 | server接受client连接的端口，ip即kafka本机ip                  |
| zookeeper.connect          | zoo1:2181               | zk连接ip:port，多个时为 ip1:port1,ip2:port2...               |
| log.retention.hours        | 168                     | 每个日志文件删除之前保存的事件，默认数据保存时间对所有topic都一样 |
| num.paritions              | 1                       | 创建topic的默认分区数                                        |
| default.replication.factor | 1                       | 自动创建topic的默认副本数量，建议设置 >= 2                   |
| min.insync.replicas        | 1                       | 当producer设置acks=-1时，min.insync.replicas 指定 replicas 的最小数目(必须确认每一个replica的写数据都是成功的)，如果该数目没有达到，producer发送消息会产生异常 |
| delete.topic.enable        | false                   | 是否允许删除主题                                             |

## Topic 和 Message Log

- topic 是一个类别的名称
  - 同类 message 发到同一个 topic 下
  - 对每个 topic，下面可以有多个 partition 日志文件

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/9B56D4F97FA44537A742A9F6DCB30C22/105121)

- paritition 是一个有序的 message 序列
  - 这些 message 按序添加到 `commit log` 文件中
  - 每个 partition 中的 message 都有一个唯一的编号，称之为 `offset`
- 每个 partition，都对应一个 commit log 文件
  - 一个 partition 中的 message 的 offset 都是唯一的
  - 但不同的 partition 中的 message 的 offset 可能相同

- kafka 一般不会删除 message，无论 message 是否被 consume
  - 会根据配置的 log.retention.hours(日志保留时间) 来确认 message 多久被删除
  - 默认保留最近一周的日志消息
  - 性能和保留 message 数据量大小没有关系
- 每个 consumer 都是基于自己在 commit log 中的消费进度(offset) 进行工作
  - 消费 offset 由 consumer 自己维护
  - 一般逐条消费 commit log 中的 message，可以通过指定 offset 重复消费某些 message、或跳过某些 message
  - consumer 对 kafka集群没什么影响

### 日志文件

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/98C8A2C959E2420FA8E7FC0E578398B9/105146)

- 消息日志文件，主要存在分区文件夹里的以 log 结尾的日志文件里
- 如下所示，表示 test1 topic 对应 partition 0 的消息日志

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/B194884946054FAF8B20AA18D827F75A/105154)

### 增加 topic 的分区数量

- 目前 kafka 不支持减少分区

```shell
# 由之前的5个partition 增加到 6个
/opt/kafka_2.13-2.7.1/bin/kafka-topics.sh -alter --topic test1 --partitions 6 --zookeeper 172.17.0.1:2181
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633762709461.png)

### topic、partiton、broker的理解

- 一个 topic 代表逻辑上的一个业务数据集，可以处理任意数量的数据
  - 比如：海量用户操作topic
- 数据量过大，比如一个 topic 里的 message 达到 10TB 时
  - 可以用 partition 分片存储数据
  - 不同 partition 在不同机器上，每台机器都运行一个 kafka 的进程 broker
- 其实就是类比 es 数据分片、redis 集群数据分槽、MySQL数据分库分表..

## 集群测试

### 容错性测试

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763444977.png)

#### 模拟节点宕机

- 如上，现在 topic1 的 partition0 的 leader 节点是 1003，将它的容器 stop 

```shell
docker stop kafka3
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763562314.png)

- 如上，可以看到 leader 已经变成了 1001，Isr 中，也没有了 1003
  - Leader 的选举是从 Isr(in sync replica) 中进行的

- 此时依然可以生产、消费消息

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763747328.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763757204.png)

- 查看 zk 中 topic partition 对应的 leader 信息

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763839637.png)

- kakfa 很多集群关键信息都存在 zk 里，保证自己的无状态，从而使集群水平扩容非常方便

#### 模拟节点恢复

```shell
# 重启 kafka3 节点
docker start kafka3
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633763996760.png)

- 如上，leader节点不会改变，Isr 中 1003 节点恢复

### 集群消费

- log 的 partition 分布在 kafka集群中不同的 broker 上
  - 每个 broker 可以请求备份其他 broker 上 patition 的数据
  - kakfa 集群支持配置一个 partition 备份的数量
- 针对每个 partition
  - 都有一个 broker 起到 `leader` 的作用
    - 处理针对该 partition 的读写请求
  - 0个或多个 broker 作为 `follwers` 
    - 被动复制 leader 的结果
    - 不提供读写（为了保证多副本数据与消费的一致性）
  - 如果 leader 挂了，其中的一个 follower 将自动成为新的 leader

#### Producers

- 生产者发送消息到 topic，同时负责选择将 message 发送到 topic 的哪一个 patition
  1. 通过 `round-robin` 做简单的负载均衡
  2. 根据 message 的某个关键字来进行区分（一般使用这种方式）

#### Consumers

- 传统消息传递模式有两种：
  - queue：队列
    - 多个 consumer 从 broker 读取数据，消息只会到达一个 consumer
  - publish / subscribe：发布订阅
    - 消息会被广播给所有的 consumer
- kafka 基于这两种模式提供了消费组的概念 consumer group
  - queue：所有 consumer 位于同一个 consumer group 下
  - publish / subscribe：所有 consumer 都有自己唯一的 consumer group

![](https://note.youdao.com/yws/public/resource/e5863162eca29c2b31e8b59c9707817d/xmlnote/9699992BE45B4C21BFE829B1A4EB34D5/105066)

- 如上图：
  - 2个broker 组成 kafka集群
  - 某个 topic 共有 4个 partition（P0 - P3）位于不同 broker 上
  - 该 topic 由 2个 consumer group 消费
  - A group 有 2个 consumer 实例，B group 有 4个 consumer 实例
- 通常一个 topic 会有几个 consumer group
  - 每个 consumer group 都是一个 logical subscriber(逻辑订阅者)
  - 每个 consumer group 有多个 consumer 实例，从而达到可扩展和容灾的机制

### 消费顺序

- 一个 partition 同一时刻在一个 consumer group 中
  - 只能有一个 consumer 实例在消费，从而保证消费顺序
- 注意：consumer group 中的 consumer 实例数量
  - 不能比 topic 中的 partition 数量多
  - 否则，多出来的 consumer 消费不到消息

- kafka 只在 partiton 范围保证消息的局部顺序性
  - 不能保证 topic 多个 partition 的消息顺序性
- 如果需要 topic 全局消息顺序消费，
  - 可以 topic partition = 1
  - consumer group 的 consumer 实例 = 1
  - 但是这样影响性能，一般很少用

## Java Client

### pom

```xml
 <dependency>
   <groupId>ch.qos.logback</groupId>
   <artifactId>logback-core</artifactId>
   <version>1.1.3</version>
</dependency>
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-access</artifactId>
  <version>1.1.3</version>
</dependency>
<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.1.1</version>
</dependency>
<dependency>
  <groupId>org.apache.kafka</groupId>
  <artifactId>kafka-clients</artifactId>
  <version>2.4.1</version>
</dependency>
```

### logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration  scan="true" scanPeriod="60 seconds" debug="false">
    <contextName>logback</contextName>
    <!--输出到控制台-->
    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="logFile"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <Prudent>true</Prudent>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <FileNamePattern>
                poslog/%d{yyyy-MM-dd}/%d{yyyy-MM-dd}.log
            </FileNamePattern>
        </rollingPolicy>
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>
                %d{yyyy-MM-dd HH:mm:ss} -%msg%n
            </Pattern>
        </layout>
    </appender>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>
                %d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
            </Pattern>
        </layout>
    </appender>

    <logger name="org.apache.kafka" level="OFF">
        <appender-ref ref="STDOUT" />
    </logger>
    <root level="INFO,ERROR">
        <appender-ref ref="console" />
        <appender-ref ref="logFile" />
    </root>


</configuration>
```

### 测试消息对象

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Order {

    private Integer orderId;

    private Integer productId;

    private Integer productNum;

    private Double amount;

}
```

### 生产者

```java
public class MsgProducer {

    /**
     * topic名
     */
    private static final String TOPIC = "test1";

    /**
     * 消息数量
     */
    private static final int MSG_NUM = 5;

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Properties props = new Properties();
        // 设置 kafka 集群连接 ip:port
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "宿主机ip:9092,宿主机ip:9093,宿主机ip:9094");

        /**
         * 发出消息持久化机制参数
         * 1: ack=0, 表示 producer 不需要等待任何 broker 确认收到消息的回复，
         *      就可以继续发送下一条消息。性能最高，但也最容易丢失消息。
         *
         * 2: ack=1, 至少等待 leader 成功将数据写入本地log，但不需要等待所有 follower 成功写入，
         *      就可以继续发送下一条消息。这种情况下，如果 follower 没有成功备份数据，leader 此时又宕机，消息就可能丢失
         *
         * 3: ack=-1或all, 需要等待 min.insync.replicas(默认为1，推荐配置 >=2) 该参数配置的 replication 副本个数成功写入日志，
         *      这种策略保证只要有一个备份存活，就不会丢失数据，一般用于跟钱相关的业务
         */
        props.put(ProducerConfig.ACKS_CONFIG, "1");

        /**
         * 发送消息失败会重试，默认重试间隔 100ms
         *      重试能保证消息发送的可靠性，但也可能造成消息重复发送，比如网络抖动
         *      所以需要在 consumer 中做好消息幂等性处理
         */
        props.put(ProducerConfig.RETRIES_CONFIG, 3);

        // 重试间隔时间设置
        props.put(ProducerConfig.RECONNECT_BACKOFF_MS_CONFIG, 300);

        // 发送消息的本地缓冲区，如果设置了该缓冲区，消息会先发送到本地缓冲区，提高消息发送性能，默认值即 32MB
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        /**
         * kafka本地线程会从缓冲区取数据，批量发送到 broker
         *      设置批量发送消息的大小，默认是 16KB，就是说一个batch满了16kb就发送出去
         */
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);

        /**
         * 默认值是0，意思就是消息立即发送，但这样有点影响性能
         *      一般设置10ms左右，即 10ms内 batch 中的消息满了 16kb 就发送
         *      10ms内，batch没满16kb，也会把消息发送出去，不能让消息发送延迟时间太长
         */
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);

        // 把 message 的 key 从字符串序列化为 byte 数组
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // 把 message 的 value 从字符串序列化为 byte 数组
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        Producer<String, String> producer = new KafkaProducer<>(props);

        CountDownLatch latch = new CountDownLatch(MSG_NUM);
        for (int i = 1; i <= MSG_NUM; i++) {
            Order order = new Order(i, 100 + i, 1, 100.00);

            // 指定发送分区
//            ProducerRecord<String, String> partitionRecord = new ProducerRecord<>(TOPIC, 0, order.getOrderId().toString(), JSONUtil.toJsonStr(order));

            // 未指定发送分区, 分区计算公式: hash(key) % partitionNum
            ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC, order.getOrderId().toString(), JSONUtil.toJsonStr(order));

            RecordMetadata metadata = producer.send(record).get();
            System.out.println("同步发送消息结果, topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());

            // 异步回调发送消息
//            producer.send(record, (metadata1, exception) -> {
//                if (exception != null) {
//                    System.out.println("消息发送失败, " + exception.getMessage());
//                    return;
//                }
//                System.out.println("异步发送消息结果, topic: " + metadata.topic() + ", partition: " + metadata.partition() + ", offset: " + metadata.offset());
//                latch.countDown();
//            });
        }
        latch.await(5, TimeUnit.SECONDS);
        producer.close();

    }

}
```

### 消费者

```java
public class MsgConsumer {

    /**
     * topic
     */
    private static final String TOPIC = "test1";

    /**
     * 消费组
     */
    private static final String CONSUMER_GROUP = "testGroup";

    public static void main(String[] args) {
        Properties props = new Properties();
        // 设置 kafka 集群连接 ip:port
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "宿主机ip:9092,宿主机ip:9093,宿主机ip:9094");

        // 设置消费组
        props.put(ConsumerConfig.GROUP_ID_CONFIG, CONSUMER_GROUP);

        // 是否自动提交offset，默认true
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false");

        // 自动提交offset的间隔时间
//        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");

        /**
         * 当消费主题的是一个新的消费组，或者指定offset的消费方式，offset不存在，那么应该如何消费
         *      latest(默认): 只消费自己启动之后发送到 topic 的消息
         *      earliest: 第一次从头开始消费，以后按照消费offset记录继续消费，这个需要区别于consumer.seekToBeginning(每次都从头开始消费)
         *
         */
//        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        /**
         * consumer 给 broker 发送心跳的间隔时间
         *      broker接收到心跳，如果此时有 rebalance 发生会通过心跳响应
         *      将 rebalance 方案下发给 consumer
         *      这个时间可以稍微设置的短一点
         */
        props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 1000);

        /**
         * broker多久收不到consumer的心跳，就认为它宕机了，会将其踢出消费组
         *      对应的 partition 也会被重新分配给其他 consumer
         *      默认是 10s
         */
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 10 * 1000);

        // 消费端一次拉取消息的最大条数
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);

        // 如果两次poll操作之间的间隔时间，超过了该设置时间
        // broker就会将该 consumer 踢出 consumer group，将分区分配给别的 consumer 消费
        props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 30 * 1000);

        // 把 message 的 key 从 byte 数组反序列化为字符串
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        // 把 message 的 value 从 byte 数组反序列化为字符串
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        Consumer<String, String> consumer = new KafkaConsumer<>(props);
        // 监听topic
        consumer.subscribe(Collections.singletonList(TOPIC));

//        // 消费指定分区
//        List<TopicPartition> partitions = Collections.singletonList(new TopicPartition(TOPIC, 0));
//        consumer.assign(partitions);
//        // 消息回溯消费
//        consumer.seekToBeginning(partitions);
//        // 指定offset消费
//        consumer.seek(new TopicPartition(TOPIC, 0), 10);
//
//        // 从指定时间点开始消费
//        List<PartitionInfo> partitionInfos = consumer.partitionsFor(TOPIC);
//        // 从1小时前开始消费
//        long fetchDateTime = System.currentTimeMillis() - 1000 * 60 * 60;
//        Map<TopicPartition, Long> map = partitionInfos.stream().collect(Collectors.toMap(k -> new TopicPartition(TOPIC, k.partition()), v -> fetchDateTime));
//        Map<TopicPartition, OffsetAndTimestamp> parMap = consumer.offsetsForTimes(map);
//        parMap.forEach((k, v) -> {
//            if (k == null || v == null) {
//                return;
//            }
//            System.out.println("partition: " + k.partition() + ", offset: " + v.offset());
//            // 根据消费里的 timestamp 确定 offset
//            consumer.assign(Collections.singletonList(k));
//            consumer.seek(k, v.offset());
//        });

        while (true) {
            // poll() 是拉取消息的长轮询
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
            if (CollUtil.isNotEmpty(records)) {
                records.forEach(v -> System.out.println("收到消息, partition: " + v.partition() + ", offset: " + v.offset() + ", key: " + v.key() + ", value: " + v.value()));

                // 手动同步提交offset，当前线程会阻塞，直到 offset 提交成功，一般使用同步提交
                consumer.commitSync();

                // 手动异步提交offset，当前线程不会阻塞，可继续处理后面的逻辑
//                consumer.commitAsync((offsets, exception) -> {
//                    if (exception != null) {
//                        System.out.println("提交失败, offset: " + offsets + "原因: " + exception.getMessage());
//                    }
//                });
            }
        }
    }

}
```

## SpringBoot整合kafka

### pom

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

### yml

```yaml
server:
  port: 8088

spring:
  kafka:
    bootstrap-servers: 宿主机ip:9092,宿主机ip:9093,宿主机ip:9094
    # 生产者设置
    producer:
      # 消费发送失败后的重试次数
      retries: 3
      batch-size: 16384
      buffer-memory: 33554432
      acks: 1
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
    # 消费者设置
    consumer:
      enable-auto-commit: false
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
    listener:
      # RECORD: 当每一条记录 被消费者监听器处理之后提交
      # BATCH: 当每一批poll()的数据 被消费者监听器处理之后提交
      # TIME: 当每一批poll()的数据 ... 距离上次提交时间大于 TIME 时提交
      # COUNT: 当每一批poll()的数据 ... 被处理 record 数量大于等于 COUNT 时提交
      # COUNT_TIME: time 或 count 有一个条件满足时提交
      # MANUAL: 当每一批poll()的数据 ... 手动调用Acknowledgment.acknowledge()后提交
      # MANUAL_IMMEDIATE: 手动调用Acknowledgment.acknowledge()后立即提交
      ack-mode: MANUAL_IMMEDIATE
```

### 生产者

```java
@RestController
public class ProducerController {

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @RequestMapping("send")
    public void send() {
        kafkaTemplate.send("test1", 0, "key", "kafkaTemplate发送消息");
    }

}
```

### 消费者

```java
@Component
public class MyConsumer {

    /**
     * @KafkaListener(groupId = "testGroup", topicPartitions = {
     *             @TopicPartition(topic = "topic1", partitions = {"0", "1"}),
     *             @TopicPartition(topic = "topic2", partitions = {"0"}, partitionOffsets = @PartitionOffset(partition = "1", initialOffset = "100"))
     *     }, concurrency = "6")
     *
     * concurrency 就是同组下的消费者个数，即并发消费数，必须小于等于分区总数
     */
    @KafkaListener(topics = "test1", groupId = "testGroup")
    public void listenTestGroup(ConsumerRecord<String, String> record, Acknowledgment ack) {
        System.out.println("test1Group收到消息");
        System.out.println(record + ", "  + record.value());
        // 手动提交offset
        ack.acknowledge();
    }

    /**
     * 配置多个消费组, 上面是 testGroup, 下面是 test2Group
     */
    @KafkaListener(topics = "test1", groupId = "test2Group")
    public void listenTest2Group(ConsumerRecord<String, String> record, Acknowledgment ack) {
        System.out.println("test2Group收到消息");
        System.out.println(record + ", "  + record.value());
        // 手动提交offset
        ack.acknowledge();
    }

}
```

## Kafka设计原理详解

### Kafka Controller

- Kafka集群中，会有一个或多个broker
  - 其中一个 broker 会被选举为控制器，即 kafka controller
  - 它负责管理集群中所有分区和副本的状态
- 当某个分区的 leader 副本出现故障时
  - 由该控制器负责为该分区选举新的 leader 副本
- 当检测到某个分区的 Isr 集合发生变化时
  - 由该控制器负责通知所有 broker 更新其元数据信息
- 当使用 kafka-topics.sh 脚本，为某个 topic 添加分区数量时
  - 由该控制器负责让 新分区 被其他节点感知到

#### controller的职责

1. 监听 broker 相关的变化
   1. 为 zk 中的 /brokers/ids 节点添加 BrokerChangeListener，用来处理 broker 增减的变化
2. 监听 topic 相关变化
   1. 为 zk 中的 /brokers/topics 节点添加 TopicChangeListener，用来处理 topic 增减的变化
   2. 为 zk 中的 /admin/delete_topics 节点添加 TopicDeletionListener，用来处理删除 topic 的动作
3. 从 zk 中读取当前所有与 topic、partition、broker 有关的信息并进行相应的管理
   1. 对所有 topic 所对应的 zk 中的 /brokers/topics/[topic] 节点添加 PartitionModificationListener，用来监听 topic 中的分区分配变化
4. 更新集群的元数据信息，同步到其他普通 broker 节点中

### Controller选举机制

- Kafka集群启动时，会自动选举一台 broker 作为 controller 来管理集群
- 集群启动选举过程：
  - 每个 broker 尝试去 zk 上创建一个 /controller 临时节点
  - zk 会保证有且仅有一个 broker 能创建成功
  - 该 broker 就会成为整个集群的 controller
- controller宕机后的选举过程：
  - 当该 controller 角色的 broker 宕机，zk /controller 临时节点就会消失
  - 其他 broker 一直监听该临时节点，发现它消失后，
    - 再次竞争创建 /controller 临时节点
    - 就重复 集群启动选举 的过程了

### Partition副本选举Leader机制

1. controller 感知到分区 Leader 所在的 broker 挂了
   - controller 监听了很多 zk 节点，可以感知到 broker 存活
2. 会根据参数 unclean.leader.election.enable 的值
   1. false：controller 会从 Isr 列表里，挑一个 broker 作为 leader
      1. 第一个 broker 最先放进 Isr 列表，可能是同步数据最多的副本
   2. true：代表 Isr 列表里，所有副本都挂了的时候，可以在 Isr 列表外的副本中选 leader
      1. 这种设置，可以提高可用性，但是选出的新 leader 有可能数据少很多

- 副本进入 Isr 列表的两个条件：
  1. 副本节点不能产生分区，必须能与 zk 保持会话、跟 leader 网络连通
  2. 副本能复制 leader 上的所有写操作，并且不能落后太多
     1. 与 leader 同步滞后的副本，是由 replica.lag.time.max.ms 配置决定的
     2. 超过该时间都没有跟 leader 同步过一次的副本，会被踢出 Isr 列表

### Consumer消费消息的offset记录机制

- 每个 consumer 会定期将自己消费分区的 offset
  -  提交给 kafka 内部topic：`_consumer_offsets`
- 提交的时候：
  - key：consumerGroupId + topic + 分区号
  - value：当前 offset 的值
- kafka 会定期清理 topic 里的消息，最后就保留最新的那条数据
- 因为 `_consumer_offsets` 可能会接收高并发请求
  - Kafka 默认给它分配了 `50` 个分区，通过加机器的方式提高并发能力
  - 可以通过 offsets_topic_num_partitions 设置
- 通过如下公式可以选出 consumer 消费的 offset 要提交到哪个分区
  - hash(consumerGroupId) % _consumer_offsets 的分区数

### Consumer Rebalance机制

- rebalance：即 consumer group 中的 consumer 数量发生变化，或消费的 partiton 数发生变化，kafka就会重新分配 consumer 和 partition 的关系
  - 比如：consumer group 中的某个 consumer 挂了
    - Kafka 会自动把分配给它的 partition 交给其他 consumer
  - 它又重启了，Kafka 又会把一些 partition 重新分配给 它
- 注意：
  - rebalance 只针对 subscribe 这种不指定分区消费的情况
  - 如果通过 assign 指定分区了，kafka就不会进行 rebalance
- 以下情况可能会触发 consumer rebalance
  1. consumer group 中的 consumer 数量发生变化
  2. 动态给 topic 增加了 partition
  3. consumer group 订阅了更多的 topic
- rebalance 过程中，consumer 无法从 kafka 消费消息，这对 kafka 的 tps 有影响
  - 如果 kafka 集群节点较多，如几百个，那重平衡可能会耗时极多
  - 所以要尽量避免在系统高峰期的 重平衡 发生

#### rebalance 分区分配策略

- 举例：某 topic 有 10个 partition（0-9），有3个 consumer

- 通过 partition.assignment.strategy 来设置
  - range(默认)：按照分区序号排序
    - 假设 n = partition 数 / consumer 数 = 3
    - m = partiton 数 % consumer 数 = 1
    - 那前 m 个 consumer 每个分配 n + 1 个 partition,
    - 后面（consumer 数 - m）个 consumer 每个分配 n 个 partition
    - 即 0-3、4-6、7-9
  - round-robin：轮询分配
    - 即 0、3、6、9
    - 1、4、7
    - 2、5、8
  - sticky：初始分配类似 round-robin，rebalance时，需要保证两个原则
    1. 分区的分配要尽可能均匀
    2. 分区的分配尽可能与上次分配的保持相同
    3. 以上两点发生冲突时，1 要优先于 2
       - 举例：对 range 的 rebalance 用 sticky 时，第3个 consumer 挂了
       - 此时 consumer1 就是 0-3、7
       - consumer2 就是4-6、8、9

#### rebalance 过程

- consumer group 加入新 consumer 时
  - consumer、consumer group、组协调器之间会经历以下几个阶段

![](https://note.youdao.com/yws/public/resource/d9fed88c81ff75e6c0e6364012d19fef/xmlnote/3A5970DAF50346389BA375660AD69D7A/105392)

##### 第一阶段：选择组协调器

-  `组协调器：GroupCoordinator`：每个 consumer group 都会选择一个 broker 作为自己的 GroupCoordinator
  - 负责监控该 consumer group 的所有 consumer 的 心跳
  - 以及判断是否宕机、然后开启 consumer rebalance
- consumer group 中的每个 consumer 启动时，
  - 都会向 kafka 集群中的某个节点发送 FindCoordinatorRequest 请求
  - 来查找对应的 GroupCoordinator，并跟它建立网络连接
- GroupCoordinator 选择方式：
  - consumer 消费的 offset 要提交到 _consumer_offsets 的哪个 partition
  - 该分区 leader 对应的 broker 就是 consumer group 的 coordinator

##### 第二阶段：加入消费组

- consumer 向 GroupCoordinator 发送 JoinGroupRequest 请求，并处理响应
  - GroupCoordinator 从一个 consumer group 中
    - 选择第一个加入该 group 的 consumer 作为 leader(消费组协调器)
    - 把 consumer group 情况发给该 leader
    - 该 leader 负责制定分区方案

##### 第三阶段：SYNC GROUP

- consumer leader 通过将 GroupCoordinator 发送 SyncGroupRequest
  - GroupCoordinator 将分区方案下发给各个 consumer
  - consumer 根据制定分区的 leader broker 进行网络连接以及消息消费

### Producer发布消息机制

#### 1. 写入方式

- producer 采用 push 模式，将消息发布到 broker
- 每条消息都被 append 到 partition 中
- 顺序顺序写磁盘（效率比随机写内存要高，保障kafka吞吐率）

#### 2. 消息路由

- producer 发送消息到 broker 时，会根据分区算法选择将其存储到哪一个 partition
- 路由机制：
  1. 指定了 partition，则直接使用
  2. 未指定 partition，但指定 key，通过对 key 的 value 进行 hash 选出一个 partition
  3. partition 和 key 都未指定，使用轮询选出一个 partition

#### 3. 写入流程

1. producer 先从 zk 的 /brokers/.../state 节点找到该 partition 的 leader
2. producer 将消息发送给该 leader
3. leader 将消息写入本地 log
4. followers 从 leader pull 消息，写入本地 log 后，向 leader 发送 ack
5. leader 收到所有 Isr 中的 replica 的 ack 后
   1. 增加 HW（高水位，high watermark，最后 commit 的 offset）
   2. 并向 producer 发送 ack

![](https://note.youdao.com/yws/public/resource/d9fed88c81ff75e6c0e6364012d19fef/xmlnote/53568FAEF32B43369F96E98E097D0895/82535)

#### HW与LEO详解

- HW：一个 partition 对应的 Isr 中最小的 LEO（log-end-offset）
  - consumer 最多只能消费到 HW 所在的位置
  - 每个 replica 都有 HW，leader 和 follower 各自负责更新自己的 HW 的状态
- 对于 leader 新写入的消息，consumer 不能立即消费
  - leader 会等待该消息被所有 Isr 中的 replicas 同步后，更新 HW
  - 此时才能被 consumer 消费
- 这保证了如果 leader 所在的 broker 宕机，该消息仍能从新选举出的 leader 获取
- 对于来自内部 broker 的读取请求，没有 HW 的限制

#### producer 生产消息至 broker后，Isr、HW、LEO的流转过程图

![](https://note.youdao.com/yws/public/resource/d9fed88c81ff75e6c0e6364012d19fef/xmlnote/94D2D8CC6AF34C40ABE4AF815F9DE93C/83193)

- 由此可见，**kafka的复制机制既不是完全的同步复制，也不是单纯的异步复制**
  - 同步复制：
    - 要求所有能工作的 follower 都复制完，该消息才会被 commit
    - 极大影响吞吐率
  - 异步复制：
    - follower 异步从 leader 复制，数据只要被 leader 写入 log 就被认为已经 commit 了
    - 这种情况下，如果 follower 都没复制完，leader 宕机，就丢失数据了
- kafka 使用 `Isr` 的方式，很好的均衡了 `确保数据不丢失` 以及 `吞吐率`
- 回顾 producer 对消息持久化机制参数 acks 的设置
  - 结合 HW 和 LEO 看 acks = 1 的情况

![](https://note.youdao.com/yws/public/resource/d9fed88c81ff75e6c0e6364012d19fef/xmlnote/040FAAC6C2264B90BD5A9FBBD138073A/83205)

## 日志分段存储

- kafka 一个 partiton 的消息数据，对应存储在一个文件夹下
  - 以 topic + partition 命名
- 消息在 partition 中是 `分段存储(segment)`
  - 每个段的消息都存储在不一样的 log 文件里
  - 方便 old segment file 快速删除
- kafka 规定一个 segment 的 log 文件最大为 1g
  - 该限制目的是为了方便将 log 文件加载到内存中操作

```apl
# 部分消息的offset索引文件，kafka每次往分区发4K(可配置)消息就会记录一条当前消息的offset到index文件，
# 如果要定位消息的offset会先在这个文件里快速定位，再去log文件里找具体消息
00000000000000000000.index
# 消息存储文件，主要存offset和消息体
00000000000000000000.log
# 消息的发送时间索引文件，kafka每次往分区发4K(可配置)消息就会记录一条当前消息的发送时间戳与对应的offset到timeindex文件，
# 如果需要按照时间来定位消息的offset，会先在这个文件里查找
00000000000000000000.timeindex

00000000000005367851.index
00000000000005367851.log
00000000000005367851.timeindex

# 9936472 代表该日志段文件里包含的起始 offset，即该分区里至少写入近千万数据
00000000000009936472.index
00000000000009936472.log
00000000000009936472.timeindex
```

- kafka broker 中 log.segment.bytes 参数，限制了每个日志段文件的大小
  - 最大就是 1GB
- 一个 segment 满了，就自动开一个新的 segment 来写
  - 避免单个文件过大，影响文件的读写性能
  - 这个过程叫做 `log rolling`
- 正在被写的 segment，叫 `active log segment`

## zk节点数据图

![](https://note.youdao.com/yws/public/resource/d9fed88c81ff75e6c0e6364012d19fef/xmlnote/2F76FF53FBF643E785B18CD0F0C2D3D2/83219)

## kafka-manager

[Github官方文档](https://github.com/yahoo/CMAK)

- 安装上面的 docker-compose.yaml 文件已经加入该服务
- 访问：宿主机ip:9000

## 线上环境规划示例

![](https://note.youdao.com/yws/public/resource/1ce32fe44f9326456e98b6e554bf6245/xmlnote/718F339245494363B9B921A103773330/105727)

### JVM参数设置示例

- kafka 是 scala 语言开发，运行在 JVM 上
- 修改 bin/kafka-start-server.sh 中的 JVM 设置，假设机器 32G 内存

```apl
export KAFKA_HEAP_OPTS="-Xmx16G -Xms16G -Xmn10G -XX:MetaspaceSize=256M -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M"
```

- 大内存情况可以使用 G1，因为年轻代内存占比比较大
  - 用 G1 可以设置 GC 最大停顿时间，不至于一次 minor gc 就  STW 很长时间
- 像 kafka、rocketMQ、es 这类中间件
  - 写数据到磁盘都会用到操作系统的 page cache
  - 所以 jvm 内存不能占满，要给 os 的缓存留出几个G

## 线上问题及优化

### 1. 消息丢失情况

#### 发送端

- 可能发生消息丢失的情况：(acks参数设置意义，在本文检索就找得到)
  - acks=0
  - acks=1
  - acks=-1或all、而 min.insync.replicas=1(默认)
    - 这种情况和 acks=1 是一样类似的效果
- 保证消息不丢失：
  - acks=-1或all、而 min.insync.replicas >= 2

#### 消费端

- 可能发生消息丢失的情况：
  - 配置的自动提交
    - 拿到消息 & 自动提交，还没来得及消费就宕机了，自然会丢失消息
- 保证消息不丢失：
  - 配置成手动提交，消费完消息、做完业务操作后再提交

### 2. 消息重复消费

#### 发送端

- 配置了重试机制，发生网络抖动之类问题，导致发送端超时
  - broker 可能已经接受到消息了
  - 但 producer 还会重新发送消息

#### 消费端

- 配置了自动提交，刚拉取了一批数据 & 处理了一部分
  - 还没来得及提交，宕机了
  - 下次重启又会拉取相同的一批数据重复处理

#### 解决方法

[advanced - java 参考链接](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-that-messages-are-not-repeatedly-consumed.md)

- 一般消费端都要做 `消费幂等` 处理
- 插库最常见的幂等处理就是：
  - 生产端消息对象增加一列 唯一id（uuid、雪花主键...），添加唯一索引
  - 生产端就插库、消费端判断 唯一id + 状态
  - 或 消费端拿到数据插库，先拿唯一id逻辑判断，唯一索引保底

### 3. 消息乱序

[advanced - java 参考链接](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/how-to-ensure-the-order-of-messages.md)

#### 发送端

- producer 配置了重试机制，发送 1、2、3 消息
  - 1 超时，2、3 发送成功
  - 重试发送 1 成功后，broker 里顺序就变成了 2、3、1

##### 解决方案

- 通常将指定 key 的消息发送到同一个分区，可以保证消息的局部有序性
- 通过配置 max.in.flight.requet.per.connection 指定
  - 发送阻塞前对于每个连接，正在发送但是发送状态未知的最大消息数量
  - 设置为1，保证在后一条消息发送前，前一条的消息状态已经是可知的

#### 消费端

1. 一个 topic、一个 partition、一个 consumer
   1. consumer 内部单线程消费
   2. 单线程吞吐量太低，一般不用这个方案
2. 写 N 个内存队列，具有相同 key 的数据都到同一个 内存队列
   1. 启用 N 个线程，每个线程对应消费一个内存队列，即：
      1. 同 key 单线程顺序消费
      2. 不同 key 在不同内存队列，由不同单线程顺序消费

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1634017339438.png)

### 4. 消息积压

[advanced-java 参考链接](https://github.com/doocs/advanced-java/blob/main/docs/high-concurrency/mq-time-delay-and-expired-failure.md)

#### 问题原因

1. producer 生产消息速度 远远大于 consumer 消费速度
2. consumer 消费 bug、一直消费不成功，broker 就积压大量未消费的消息

#### 解决方案

1. 修复问题后，consumer 临时扩容，例如：增加十倍consumer资源，快速消费积压消息
2. 消费失败的，转发至其他队列（例如：死信队列），后续再人工补偿

### 5. 延时队列

#### 需求场景

- 创建订单，15分钟后仍未支付，则自动关闭该订单

#### 解决方案

1. 创建一个专门用于轮询 延时任务的 topic
2. producer 发送消息，消息添加算好的到期执行时间属性、目标业务topic
   1. 如，12:00发送，15分钟后执行，消息中就带着 12.15 的属性
   2. 消息目标发往 order topic，消息中就带着 order 的属性
3. 轮询 topic 中，定时每秒 对所有延时任务进行轮询判断
   1. 到期执行时间 >= 当前时间时，就转发到目标topic

### 6. 消息回溯

#### 需求场景

- 因程序bug导致 12:00 - 14:00 该时间段的消息消费错误
  - 当 bug 修复后，需要对该时间段的消息重新消费

#### 解决方案

- 使用 consumer 中 offsetsForTimes、seek 等方法
  - 指定从某个 offset 偏移的消息开始消费
  - 具体使用参考上面的代码实例

### 7. 分区数与吞吐量的关系

#### 问题

- topic 的 partition 越多，吞吐量就越高吗？

#### kafka压测工具

- 用如下工具测试不同 partition 数下的吞吐量

```shell
# 往test里发送一百万消息，每条设置1KB
# throughput 用来进行限流控制，当设定的值小于 0 时不限流，当设定的值大于 0 时，当发送的吞吐量大于该值时就会被阻塞一段时间
bin/kafka-producer-perf-test.sh --topic test --num-records 1000000 --record-size 1024 --throughput -1 --producer-props bootstrap.servers=宿主机ip:9092 acks=1
```

![](https://note.youdao.com/yws/public/resource/1ce32fe44f9326456e98b6e554bf6245/xmlnote/E636598909724676BA68B98224DEE864/83466)

#### 结论

- partition 数到达某个阈值后、再设高，吞吐量反而开始下降
- 一般情况下，建议 partition 数跟 kafka 集群中 broker 数保持一致

#### 注意

- partiton 数设置过大，比如 10000，可能会报：
  - java.io.IOException : Too many open files
  - 一种常见的 Linux 系统错误，意味着 文件描述符不足
  - 一般默认都是 1024，可以用 ulimit -n 命令查看
  - 可以用 ulimit -n xxx(比如 10000) 调整该值的大小

### 8. 消息传递保障

#### 三种传递机制

- at most once：消费者最多收到一次消息，0 - 1
  - acks = 0 可以实现
- at least once：消费者最少收到一次消息，1 - n
  - acks = -1或all 可以实现
- exactly once：消费者刚好收到一次消息
  - `acks = -1或all + 消费者幂等` 可以实现
  - kafka producer 幂等性 可以实现

#### kakfa producer 幂等性

- kafka producer 幂等性，可以保证 producer 重试导致的重复发送消息
  - 只会被接收一次
- 在 producer 添加如下属性开启：
  - props.put("enable.idempotence", true)
  - 默认为 false

##### 原理

- producer 每次发消息生成 `PID` 和 `Sequence Number` 至 broker
  - broker 将 PID、Sequence Number 和 消息绑定存起来
  - producer 再发消息，broker 会根据这两个属性判断，相同则拒收
- PID：
  - 每个新 producer 在初始化的时候，会分配一个唯一的 PID
  - 对用户完全透明
  - producer 重启则生成新的 PID
- Sequence Number：
  - 对于每个 PID，该 producer 发送到每个 partition 的数据都有对应的序列号
  - 序列号从 0 开始单调递增

### 9. kafka事务

#### 需求场景

- 订单支付成功回调，往不同 topic 发消息，监听业务 改状态、发积分、推短信...

- kafka 事务只能保障 `一次发送多条消息，要么同时成功，要么同时失败（支持多topic）`
  - 想像 rocketMQ 一样保障 db 和 mq 的事物一致性，就需要自己额外开发

```java
Properties props = new Properties();

props.put("bootstrap.servers", "localhost:9092");
props.put("transactional.id", "my-transactional-id");

Producer<String, String> producer = new KafkaProducer<>(props, new StringSerializer(), new StringSerializer());

// 初始化事务
producer.initTransactions();

try {
  // 开启事务
  producer.beginTransaction();
  for (int i = 0; i < 100; i++){
    // 发到不同的主题的不同分区
    producer.send(new ProducerRecord<>("hdfs-topic", Integer.toString(i), Integer.toString(i)));
    producer.send(new ProducerRecord<>("es-topic", Integer.toString(i), Integer.toString(i)));
    producer.send(new ProducerRecord<>("redis-topic", Integer.toString(i), Integer.toString(i)));
  }
  // 提交事务
  producer.commitTransaction();
} catch (ProducerFencedException | OutOfOrderSequenceException | AuthorizationException e) {
  // We can't recover from these exceptions, so our only option is to close the producer and exit.
  producer.close();
} catch (KafkaException e) {
  // For all other exceptions, just abort the transaction and try again.
  // 回滚事务
  producer.abortTransaction();
}
producer.close();
```

### 10. kafka高性能原因

1. 磁盘顺序读写
   1. 消息不能修改、不会从文件中间删除，保证 磁盘顺序读
   2. 消息写入都是追加到文件末尾，保证 磁盘顺序写
2. 数据传输的零拷贝
   1. ![](https://note.youdao.com/yws/public/resource/1ce32fe44f9326456e98b6e554bf6245/xmlnote/C4EFFC3047644E5AB2A15B872E90B4EC/105677)
3. 读写数据的 batch 批处理、以及压缩传输

