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
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:9092
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
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:9093
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
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:9094
      KAFKA_ADVERTISED_PORT: 9094
      KAFKA_ZOOKEEPER_CONNECT: zoo1:2181,zoo2:2181,zoo3:2181
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

