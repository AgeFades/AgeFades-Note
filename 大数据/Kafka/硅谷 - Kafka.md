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

## Kafka 生产者

### 分区策略

### 分区的原因

```shell
# 方便在集群中扩展
	# 每个 partition 可以通过调整以适应它所在的机器
	
	# 而一个 topic 又可以由多个 partition 组成，因此整个集群就可以适应任意大小的数据了
	
# 可以提高并发，因为可以以 partition 为单位读写了
```

#### 分区的原则

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

#### 图解

![UTOOLS1579485116060.png](http://yanxuan.nosdn.127.net/e11aa7e45a05c1f590bef12439e50370.png)

#### 副本数据同步策略

| 方案                         | 优点                                                    | 缺点                                                 |
| ---------------------------- | ------------------------------------------------------- | ---------------------------------------------------- |
| 半数以上完成同步，就发送 ack | 延迟低                                                  | 选举新的leader时，容忍n台节点的故障，需要2n+1 个副本 |
| 全部完成同步，才发送ack      | 选举新的leader 时，容忍 n 台节点的故障，需要 n+1 个副本 | 延迟高                                               |

```shell
# Kafka 选择了第二种方案，原因如下:
	
# 同样为了容忍 n 台节点的故障，
	# 第一种方案需要 2n+1 个副本，而第二种方案只需要 n+1 个副本
	# 而 Kafka 的每个分区都有大量的数据，第一种方案会造成大量数据的冗余

# 虽然第二种方案的网络延迟会比较高，但网路延迟对 Kafka 的影响较小
```

#### ISR

```shell
# 采用第二种方案之后，设想以下情景:
	# leader 收到数据，所有 folllower 都开始同步数据，
	# 但有一个 follower 因为某种故障，迟迟不能与 leader 进行同步，
	# 那 leader 就要一直等下去，知道它同步完成，才能发送 ack，
	# 这个问题怎么解决呢？
```

```shell
# leader 维护了一个动态的 in-sync replica set(ISR)
	# 意为和 leader 保持同步的 follower 集合
	
# 当 ISR 中的 follower 完成数据的同步之后，
	# leader 就会给 follower 发送 ack，
	# 如果 follower 长时间未向 leader 同步数据，
	# 则该 follower 将被踢出 ISR，
	# 该时间阈值由 replica.lag.time.max.ms 参数设定
	# leader 发生故障之后，就会从 ISR 中选举新的 leader。
	
```

#### ack 应答机制

```shell
# 对于某些不太重要的数据，对数据的可靠性要求不是很高，
	# 能够容忍数据的少量丢失，所有没必要等 ISR 中的 follower 全部接收成功

# 所以 Kafka 为用户提供了三种可靠性级别，
	# 用户根据对可靠性和延迟的要求进行权衡，选择如下的配置
	
# acks 参数配置:
	# 0: producer 不等待 broker 的 ack，这一操作提供了一个最低的延迟，broker 一接收到还没有写入磁盘就已经返回，当 broker 故障时有可能丢失数据。
	
	# 1: producer 等待 broker 的 ack，partition 的 leader 落盘成功后返回 ack，如果在 follower 同步成功之前 leader 故障，那么就会丢数据。
	
	# -1(all): producer 等待 broker 的 ack，partition 的 leader 和 follower 全部落盘成功后才返回 ack。但是如果在 folllower 同步完成后，broker 发送 ack 之前，leader 发生故障，那么会造成数据重复。
```

#### 故障处理细节

##### Log 文件中的 HW 和 LEO

![UTOOLS1579486376439.png](http://yanxuan.nosdn.127.net/bfb7d48d27c64c9325f4340847b8936f.png)

```shell
# LEO: 每个副本最大的 offset

# HW: 消费者能见到的最大的 offset，ISR 队列中最小的 LEO
```

##### follower 故障

```shell
# follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，
	# follower 会读取本地磁盘记录的上次的 HW，
	# 并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。
	# 等该 follower 的 LEO 大于等于该 Partition 和 HW，
	# 即 follower 追上 leader 之后，就可以重新加入 ISR 了
```

##### leader 故障

```shell
# leader 发生故障之后，会从 ISR 中选出一个新的 leader，
	# 之后，为保证多个副本之间的数据一致性，
	# 其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，
	# 然后从新的 leader 同步数据
	
# 注意: 这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复
```

### Exactly Once 语义

```shell
# 将服务器的 ACK 级别设置为 -1，可以保证 Producer 到 Server 之间不会丢失数据
	# 即 At Least Once 语义
	
# 相对的，将服务器 ACK 级别设置为 0，可以保证生产者每条消息只会被发送一次
	# 即 At Most Once 语义
	
# 但是，对于一些非常重要的信息，下游数据消费者要求数据既不重复也不丢失
	# 即 Exactly Once 语义
```

```shell
# 0.11 版本的 Kafka，引入了一项重大特性: 幂等性

# 所谓的幂等性就是指 Producer 不论向 Server 发送多少次重复数据，
	# Server 端都只会持久化一条，
	# 幂等性结合 At Least Once 语义，就构成了 Exactly Once 语义
	# 即: At Least Once + 幂等性 = Exactly Once
	
# 要启用幂等性，只需要将 Producer 的参数中 enable.idompotence 设置为 true 即可
	
# Kafka 的幂等性实现其实就是将原来下游需要做的去重放在了数据上游。
	# 开启幂等性的 Producer 在初始化的时候会被分配一个 PID，
	# 发往同一 Partition 的消息会附带 Sequence Number，
	# 而 Broker 端会对 <PID, Partition, SeqNumber> 做缓存，
	# 当具有相同主键的消息提交时，Broker 只会持久化一条。
	# 但是 PID 重启就会发生变化，同时不同的 Partition 也具有不同主键，
	# 所以幂等性无法保证跨分区跨会话的 Exactly Once。
```

## Kafka 消费者

### 消费方式

```shell
# consumer 采用 pull(拉) 模式从 broker 中读取数据。

# push(推) 模式很难适应消费速率不同的消费者，因为消息发送速率是由 broker 决定的。
	# 它的目标是尽可能以最快速度传递消息，但是容易造成 consumer 来不及处理消息。
	# 典型的表现就是:
		# 拒绝服务以及网络拥塞。
	
# 而 pull 模式则可以根据 conusmer 的消费能力以适当的速率消费消息。

# pull 模式的缺点是:
	# 如果 kafka 没有数据，消费者可能会陷入循环中，一直返回空数据。
	# 针对这一缺点, Kafka 的消费者在消费数据时会传入一个时长参数 timeout
		# 如果当前没有数据可供消费，consumer 等待 timeout 之后再返回。
```

### 分区分配策略

```shell
# 一个 consumer group 中有多个 consumer，一个 topic 有多个 partition
	# 所以必然涉及 partition 的分配问题，
	# 即确定哪个 partition 由哪个 consumer 来消费。
	
# Kafka 有两种分配策略，一是 RoundRobin，一是 Range(默认)。
```

### offset 的维护

```shell
# 由于 consumer 故障宕机恢复后，需要从故障前的位置继续消费，
	# 所以 consumer 需要实时记录自己消费到了哪个 offset。
	
# Kafka 0.9 版本之前，consumer 默认将 offset 保存在 zk 中
	# 0.9 之后，默认将 offset 保存在 Kafka 一个内置的 topic 中，
	# 该 topic 为 _cnsumer_offsets
```

### Kafka 高效读写数据

```

```

