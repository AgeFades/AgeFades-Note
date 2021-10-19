[TOC]

# 楼兰 - RocketMQ

[官方文档](http://rocketmq.apache.org)

- 本文基本从官方文档摘抄
- 摘抄的目的：加深个人印象、总结收录、便于后续翻阅

## 简介

- 阿里开源的一个消息中间件，经历淘宝双十一考验，能处理亿万级别消息
- 16年开源后捐给 Apache，是 Apache 的顶级项目

## 技术架构图

![](https://note.youdao.com/yws/public/resource/9a677e8cbccb2808802d608abe2b8a8d/BAD094A2F5B249EA87FB5B9048B62B19?ynotemdtimestamp=1634022407850)

## 基本概念

### Message Model(消息模型)

- RocketMQ主要由三部分组成：
  - producer
    - 生产消息
  - broker
    - 存储消息
    - 对应一个 RocketMQ 实例
    - 每个 broker 可以存储多个 topic 的消息
    - 每个 topic 的消息可以分片存储于不同的 broker
  - consumer
    - 消费消息
- Message Queue：
  - 用于存储消息的物理地址
  - 每个 topic 中的消息地址，存储于多个 Message Queue 中
- Consumer Group：
  - 由多个 consumer 实例构成

### Producer(生产者)

- 生产消息的业务系统，将消息发送至 broker
- RocketMQ 提供多种发送方式：
  - 同步发送
    - 需要 broker 返回确认信息
  - 异步发送
    - 需要 broker 返回确认信息
  - 顺序发送
  - 单向发送
    - 不需要 broker 返回确认消息

### Consumer(消费者)

- 消费消息的业务系统
- 提供两种消费形式：
  - 主动拉取（默认）
  - broker推送

### Topic(主题)

- 一类消息的集合，每个 topic 包含若干 message
- 一条 message 只能属于一个 topic
- 是 RockertMQ 进行消息订阅的基本单位

### Broker(代理服务器)

- 负责存储消息、转发消息
- 还存储消息相关的元数据
  - consumer group
  - 消费进度偏移 offset
  - topic
  - message queue
  - ...

### NameServer(名称服务)

- 路由消息的提供者
- producer 或 consumer 通过 NameServer 查找各 topic 对应的 broker ip列表
- 多个 NameServer 组成集群，但相互独立，没有信息交换

### Pull Consumer(拉取式消费)

- consumer 主动从 broker 拉消息
- 一旦获取批量消息，consumer 就会启动消费过程

### Push Consumer(推动式消费)

- broker 收到消息后，主动推送给 consumer
- 一般实时性较高

### Producer group(生产者组)

- 同类 producer 的集合，发送同一类消息 & 发送逻辑一致
- 如果发事务消息，且原始生产者在发送之后宕机，
  - 则 broker 会联系同一 producer group 的 其他 producer 以 提交 或 回溯 消费

### Consumer Group(消费者组)

- 同类 consumer 集合，消费同一类消息 & 消费逻辑一致
  - 所有 consumer 都必须订阅完全相同的 topic
- consumer group 实现 consumer 消费消息的：
  - 负载均衡
  - 容灾 

### 消费模式

#### Clustering(集群消费)

- 相同 consumer group 的每个 consumer 平均分摊消息

#### Broadcasting(广播消费)

- 相同 consumer group 的每个 consumer 接收全量消息

### 顺序消息

#### Normal Ordered Message(普通顺序消息)

- consumer 通过同一个 消息队列（topic 分区、称作 Message Queue）
  - 收到的消息是有序的
- 不同 消息队列 收到的消息则可能是 无序 的

#### Strictly Ordered Message(严格顺序消息)

- consumer 收到的所有消息均是 有序的

### Message(消息)

- producer 和 consumer 的最小单位
- 每条 message 必须属于一个 topic
- 每条 message 都有唯一的 message id
  - 可以携带具有业务标识的 key
- 系统提供了通过 message id 和 key 查询消息的功能

### Tag(标签)

- 给 message 打 tag，用于区分同一 topic 下不同类型的 message
  - 举例：order topic 下，pay(支付) 是一个 tag、expire(超时) 是一个 tag

## 特性

### 1. 订阅与发布

- 发布：
  - producer 向 broker 中的某个 topic 发消息
- 订阅：
  - consumer 监听某个 topic 中带有某些 tag 的消息，进行拉取消费

### 2. 消息顺序

- 需求场景举例：
  - create order、order pay、order success 
  - 需要按顺序消费的业务消息
- 顺序消息分为：
  - `全局顺序`：
    - 某个 topic 下所有消息，producer、consumer都要保证顺序
    - 适用场景：`性能要求不高、对消息顺序有严格要求`
  - `分区顺序`：
    - 部分顺序消息，只要保证每一组消息被顺序消费即可
    - 指定 topic，所有 message 都根据 sharding key 进行 区块分区
      - sharding key 是 顺序消息 中区分不同区分的 关键字段
      - 跟普通 message 的 key 是完全不同的概念
    - 同一个分区内的 message，producer、consumer保证顺序
    - 适用场景：`性能要求高、对消息顺序有部分要求`

### 3. 消息过滤

- consumer 可以根据 tag 进行消息过滤，也支持 自定义属性 过滤
- 消息过滤目前是在 broker 端实现的，
  - 优点：减少对 consumer 无用消息的网络传输
  - 缺点：增加 broker 的负担、且实现相对复杂

### 4. 消息可靠性

- 影响消息可靠性的几种情况：
  1. broker 宕机
  2. broker 异常 crash
  3. os crash
  4. 机器电源波动
  5. 机器 cpu、主板、内存 等关键硬件损坏，导致无法开机
  6. 机器磁盘设备损坏
- RocketMQ 针对以上情况：
  - 1、2、3、4，属于硬件资源可立即恢复情况
    - RocketMQ 保证消息不丢、或丢失少量数据
    - 依赖 `刷盘方式` 是 同步 还是 异步
  - 5、6 属于单点故障、且无法恢复，一旦发生，此节点消息全部丢失
    - RocketMQ 通过异步复制，可保证 99% 消息不丢
    - 通过 `同步双写` 可完全避免单点故障（影响性能，适用对可靠性要求极高的场景，例如：涉及到钱的业务）3.0 往上版本支持 `同步双写`

### 5. 至少一次

- At least once，每个消息必须投递一次
  - consumer 先 pull 消息到本地，消费完成后，返回 ack 给 broker
  - 如果没有消费完成，一定不会 ack

### 6. 回溯消息

- 适用场景：
  - 程序bug，一小时内消费消息都导致异常数据，需要修复bug后，重新消费
- RocketMQ  broker 需要给 consumer 投递成功消息后，仍保留消息
  - broker 提供可按照时间维度，来回退消费进度，精确到毫秒

### 7. 事务消息

- RocketMQ  提供 XA 的分布式事务功能，
  - 支持 应用本地事务 和 发送消息操作 定义到 `全局事务` 中
  - 要么 同时成功，要么 同时失败

### 8. 定时消息

- 即 `延迟队列`、message 到 broker 后，不会立即消费，而是等到 特定时间 才投递给 业务 topic
- broker 配置项：`messageDelayLevel`，默认支持如下 18个 level
  - 1s、5s、10s、30s
  - 1m、2m、3m、4m、5m、6m、7m、8m、9m、10m、20m、30m
  - 1h、2h
- 可以配置自定义broker 的 messageDelayLevel
- 发消息时，设置 delayLevel 等级即可：
  - msg.setDelayLevel(level)
  - level 有以下三种情况：
    - level == 0：消息为非延迟消息
    - 1 <= level <= maxLevel：消息延迟特定时间，例：level == 1，延迟 1s
    - level > maxLevel：则 level == maxLevel，例：level == 20，延迟 2h
- 定时消息暂存在 `SCHEDULE_TOPIC_XXXX`  的 topic 中
  - 并根据 delayTimeLevel 存入特定的 queue
  - queueId = delayTimeLevel - 1
  - 即一个 queue 只存相同延迟的 message，保证具有相同延迟的 message 顺序消费
  - broker 会调度消费 `SCHEDULE_TOPIC_XXXX`，将满足延时条件的 message 推到 业务 topic
- 注意：
  - 定时消息会在 第一次写入 和 调度写入业务topic 时，都会计数
  - 因此发送数量、tps都会变高

### 9. 消息重试

- consumer 消费失败后，要提供一种重试机制，让 consumer 再消费一次改消息
- 消费失败情况通常为：
  - 该消息导致的业务异常，比如插库时、唯一性校验失败了
    - 这种情况，立马重试99%也是失败
    - 最好提供一种定时重试机制，比如过 10s 再重试
  - 消费端系统异常，比如数据库挂了连不上了
    - 这种情况，跳过该消息、下条消息也会失败
    - 建议 consumer sleep 30s，再消费重试
  - 以上都是为了减轻 broker 重试消息的压力
- RocketMQ 会为`每个 consumer group` 都设置一个 topic 名称为 `"%RETRY% + consumerGroup"` 的重试队列
  - 用于暂存因 各种异常 而导致 consumer 无法正常消费的 消息
  - 考虑到 bug 修复时间，会为 重试队列 设置多个 重试级别
  - 每个 重试级别 都有与之对应的 重新投递延时
  - 重试次数越多、重新投递延时就越大
  - 其实就是 1s、3s、5s、30s... 这样子，错的次数越多、延时重试的就越多
- RocketMQ 将 重试消息 先保存至 "SCHEDULE_TOPIC_XXX" 的延迟队列 topic 中
  - 后台定时判断 delay 时间、到点的就推送至` "%RETRY% + consumerGroup"` 重试队列中

### 10. 消息重投

- producer 发 message 时
  - 同步消息失败会 重投
  - 异步消息有 重试
  - oneway没有任何保证
- 消息重投，尽可能保证消息不丢失，但可能造成 重复消费
  - producer 主动重发、consumer 负载变化，都可能导致重复消息
- 设置消息重试策略：
  - `retryTimesWhenSendFailed`：
    - 同步发送失败重投次数，`默认为2`
    - 不选上次失败的 broker、会选其他 broker
    - 超过重投次数、抛出异常，由业务自己去保证消息不丢
    - 出现 RemotingException、MQClientException 和 部分 MQBrokerException 时，会重投
  - `retryTimesWhenSendAsyncFailed`：
    - 异步发送失败重试次数
    - 只在同一个 broker 上重试，不保证消息不丢
  - `retryAnotherBrokerWhenNotStoreOK`：
    - 消息刷盘（主或备）超时、或 salve 不可用(返回状态非 SEND_OK) 时，是否尝试发送到其他 broker
    - 默认为 false
    - 看场景，很重要的消息可以设置为 true

### 11. 流量控制

- 生产者流控：broker 处理能力达到瓶颈，通过拒绝 send 请求实现流控
  - commit log文件，被锁时间超过 `osPageCacheBusyTimeOutMills(默认为1000ms)` 时，返回流控
  - `transientStorePoolEnable == true`、
    - 且 broker 为异步刷盘的主机、
    - 且 transientStorePool 中资源不足、
    - 拒绝当前 send 请求，返回流控
  - broker 每隔 10ms 检查 send 请求队列头请求 的等待时间，
    - 如果超过 `waitTimeMillsInSendQueue(默认200ms)`
    - 拒绝当前 send 请求，返回流控
  - 注意：producer 流控，不会尝试 消息重投
- 消费者瓶颈：消费能力达到瓶颈，通过 降低拉取频率 实现流控
  - consumer 本地缓存消息数 超过 `pullThresholdForQueue(默认1000)` 时
  - consumer 本地缓存消息大小 超过 `pillThresholdSizeForQueue(默认100MB)` 时
  - consumer 本地缓存消息跨度 超过 `consumerConcurrentlyMaxSpan(默认2000)` 时

### 12. 死信队列

- 消息一直发送失败，重试次数用完了还是失败，就会被发送到 死信队列
  - 死信消息：`Dead-Letter Message`
  - 死信丢列：`Dead Letter Queue`
- RocketMQ中，可以通过使用 console 控制台，手动对死信队列中的消息进行重发

## 略过的记录

- 架构、设计、配置项... 乱七八糟的
- 直接去官网翻就行了，中国人写的框架，都有中文文档

## 安装部署

