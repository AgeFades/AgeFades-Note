# 硅谷 - Kafka

## Kafka 概述

### 定义

```shell
# Kafka 是一个 分布式 的基于 发布/订阅 模式的消息队列，主要应用于大数据实时处理领域。
```

### 消息队列

#### 传统消息队列的应用场景

```shell
# 这些就不再赘述了，有过开发经验的小伙伴都大概了解过一些。
```

#### 使用消息队列的好处

```shell
# 核心六字: 异步、解耦、削峰
```

#### 消息队列的两种模式

```shell
# 点对点

# 发布/订阅
```

### Kafka 基础架构

![UTOOLS1578726621720.png](http://yanxuan.nosdn.127.net/c20df3824794a6f7e9b02c5cc6f389a9.png)

```shell
# 核心点:
	# 生产者生产消息
	# Kafka 集群管理消息
	# 消费者消费消息
	# Zookeeper 注册消息
```

#### Kafka 核心概念

##### Producer

```shell
# 消息生产者，就是 kafka broker 发消息的客户端
```

##### Consumer

```shell
# 消息消费者，向 kafka broker 取消息的客户端
```

##### Consumer Group（CG）

```shell
# 消费者组，由多个 consumer 组成。

# 消费者组内每个消费者负责消费不同分区的数据
	# 一个分区只能由一个组内消费者消费
	# 消费者组之间互不影响
	# 所有的消费者都属于某个消费者组
	# 即 消费者组是逻辑上的一个订阅者
```

##### Broker

```shell
# 一台 kafka 服务器就是一个 broker

# 一个集群由多个 broker 组成

# 一个 broker 可以容纳多个 topic
```

##### Topic

```shell
# 可以理解为一个队列

# 生产者和消费者面向的都是一个 topic
```

##### Partition

```shell
# 为了实现扩展性，一个非常大的 topic 可以分布到多个 broker 上

# 一个 topic 可以分为多个 partition

# 每个 partition 是一个有序的队列
```

##### Replica

```shell
# 副本

# 为保障集群中的某个节点发生故障时，该节点上的 partition 数据不丢失
	# 且 kafka 仍然能够继续工作

# kafka 提供了副本机制，一个 topic 的每个分区都有若干个副本
	# 一个 leader 和若干个 follower
```

##### Leader

```shell
# 每个分区多个副本的 "主"

# 生产者发送数据的对象，以及消费者消费数据的对象都是 leader
```

##### Follower

```shell
# 每个分区多个副本中的 "从"

# 实时从 leader 中同步数据，保持和 leader 数据的同步

# leader 发生故障时，某个 follower 会成为新的 follower
```

## Kafka 快速入门

### 安装部署

```shell
# 下载页面: http://kafka.apache.org/downloads.html

# 此处略...
```

### 配置文件

```shell
# config/server.properties
```

```properties
# broker 的全局唯一编号，不能重复
broker.id=0
# 是否可以删除 topic
delete.topic.enable=true
# 处理网络请求的线程数量
num.network.threads=3
# 用来处理磁盘 IO 的线程数量
num.io.threads=8
# 发送套接字的缓冲区大小
socket.send.buffer.bytes=102400
# 接收套接字的缓冲区大小
socket.receive.buffer.bytes=102400
# 请求套接字的缓冲区大小
socket.request.max.bytes=104857600
# kafka 运行日志存放的路径（实际上也是 Kafka 暂存数据的目录）
log.dirs=/opt/module/kafka/logs
# topic 在当前 broker 上的分区个数
num.partitions=1
# 用来恢复和清理 data 下数据的线程数量
num.recovery.threads.per.data.dir=1
# segment 文件保留的最长时间（单位为小时），超时将被删除
log.retention.hours=168
# 日志文件存储的最大容量（默认 1g，超过这个大小，会产生一个新的文件，新文件命名为index文件中的索引位置）
log.segment.bytes=1073741824
# 配置连接 zk 的地址（可集群,集群用逗号分隔）
zookeeper.connect=localhost:2181
```

### 配置环境变量

```shell
# vim /etc/profile
export KAFKA_HOME=/opt/module/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

## Kafka 架构深入

### Kafka 工作流程及文件存储机制

![UTOOLS1578900772377.png](http://yanxuan.nosdn.127.net/9dbcdecb6e899bfb72d5929425e08d00.png)

#### 工作流程

```shell
# Kafka 中消息是以 topic 进行分类的，生产者生产消息，消费者消费消息，都想面向 topic 的。

# topic 是逻辑上的概念，而 partition 是物理上的概念。
	# 每个 partition 对应于一个 log 文件，该 log 文件中存储的就是 producer 生产的数据
	
	# Producer 生产的数据会被不断追加到该 log 文件末端，且每条数据都有自己的 offset
	
	# 消费者组中的每个消费者，都会实时记录自己消费到了哪个 offset，以便出错恢复时，从上次的位置继续消费
```

#### 存储机制

![UTOOLS1578901120033.png](http://yanxuan.nosdn.127.net/a8f3b1aa628943978ce00bcf73d761dc.png)

```shell
# 由于生产者生产的消息会不断追加到 log 文件末尾，为防止 log 文件过大导致数据定位效率低下:
	# Kafka 采取了 分片 和 索引 机制
	
	# 将每个 partition 分为多个 segment
	
	# 每个 segment 对应两个文件 -> .index 文件 和 .log 文件
	
	# 这些文件位于一个文件夹下，该文件夹的命名规则为:
		# topic 名称 + 分区序号
		# 例如: first 这个 topic 有三个分区，则其对应的文件夹为 first-0,first-1,first-2
```

```shell
# index 和 log 文件以当前 segment 的第一条消息的 offset 命名
	# index 文件存储大量的索引信息

	# log 文件存储大量的数据
	
	# 索引文件中的元数据指向对应数据文件中 message 的物理偏移地址

# index 文件和 log 文件的结构示意图:
```

```shell
# 比如找 offset 为 3 的消息 ->
	# index 文件中二分查找法找到 3 对应 msg 的偏移量，比如是 756 ->
	# log 文件中有每条消息的固定大小，比如是 1000 ->
	# 那第三条消息的数据就是 756 - 1756 区间值
```

![UTOOLS1578901384640.png](http://yanxuan.nosdn.127.net/45df0738cb5da36200011d83e93e3eda.png)

### Kafka 生产者

#### 分区策略

##### 分区的原因

```shell
# 方便在集群中扩展
	# 每个 partition 可以通过调整以适应它所在的机器
	
	# 而一个 topic 又可以由多个 partition 组成，因此整个集群就可以适应任意大小的数据了
	
# 可以提高并发，因为可以以 partition 为单位读写了
```

##### 分区的原则

```shell
# 我们需要将 producer 发送的数据封装成一个 ProducerRecord 对象

# 指明 partition 的情况下，直接将指明的值作为 partition 值

# 没有指明 partiton 值但有 key 的情况下:
	# 将 key 的 hash 值与 topic 的 partition 数进行取余得到 partition 值
	
# 既没有 partition 值又没有 key 值的情况下:
	# 第一次调用时随机生成一个整数（后面每次调用在这个整数上自增）
	# 将这个值与 topic 可用的 partition 总数取余得到 partition 值
	# 也就是常说的 round-robin 算法
```

![UTOOLS1578905021711.png](http://yanxuan.nosdn.127.net/29973b0b311c4fd979d7b5ed1734e4a7.png)

### 数据可靠性保证

```shell
# 为保证 producer 发送的数据，能可靠的发送到指定的 topic:
	# topic 的每个 partition 收到 producer 发送的数据后 ->
	# 都需要向 producer 发送 ack（acknowledgement 确认收到）->
	# 如果 producer 收到 ack，就会进行下一轮的发送 ->
	# 否则重新发送该条数据
```

