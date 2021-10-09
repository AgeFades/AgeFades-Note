[TOC]

# 白起 - RabbitMQ

## MQ的基本概念

### 概述

- Message Queue 消息队列
- 多用于分布式系统之间进行通信				 		

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664494104.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664507632.png)

### 优势

#### 解耦

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664570396.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664590342.png)

#### 异步

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664617717.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664645797.png)

#### 削峰

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664704145.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664720295.png)

### 劣势

- 引用新组建，系统变复杂
- MQ单点故障，导致系统业务受影响
- 消息丢失、重复消费等问题

### 常见MQ产品

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664869708.png)

## RabbitMq

[官方文档](https://www.rabbitmq.com/getstarted.html)

### AMQP

- Advanced Message Queue Protocol 高级消息队列协议

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633664983234.png)

### 简介

- RabbitMQ 是 Erlang 语言开发
  - Erlang 专注于高并发、分布式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1633665058207.png)

### 相关概念

#### Broker

- 接收和分发消息的应用

#### Virtual host

- 虚拟主机，就是分组的意思
- 每个 vhost 创建自己的 exchange、queue等

#### Connection

- publisher、consumer 和 broker 之间的 tcp连接

#### Channel

- connection 内部建立的逻辑连接
- 单个线程创建单独的 channel 进行通讯，channel之间相互隔离
- AMQP method 包含 channel id，被客户端、服务端用来识别 channel
- 用于减轻每次建立 tcp connection 的开销

#### Exchange

- message 到达 broker 的第一站
  - 根据分发规则，查询匹配表中的 routing key
  - 决定分发到具体的 queue 中
- 常用类型：
  - direct（点对点）
  - topic（生产、订阅）
  - fanout（多播）

#### Queue

- message 最终到达 queue，等待被消费

#### Binding

- exchange 和 queue 的虚拟连接
  - 可包含 routing key
- 被保存至 exchange 的查询表中，用于 message 的分发

### 搭建部署



### 工作模式

#### simple

![](https://www.rabbitmq.com/img/tutorials/python-one.png)

- 一个生产者对应一个消费者，使用默认交换机

#### work queues

![](https://www.rabbitmq.com/img/tutorials/python-two.png)

- 工作队列模式
  - 多个消费者 对于 同一个消息 的关系是 **竞争**
  - 底层使用的 默认交换机（default exchange）
- 应用场景：
  - 多个短信服务消费者，监听到消息后，只需要一个节点发送短信即可

#### publish / subscribe

![](https://www.rabbitmq.com/img/tutorials/python-three.png)

- 发布订阅模式
  - 一个消息 会被 **多个消费者消费**
- 生产者发消息到交换机（类型为fanout）
  - exchange类型 决定如何处理 message
    - fanout：广播，message 转至所有绑定了该 exchange 的 queue（这里讲的就是fanout）
    - direct：定向，message 转至符合指定 routing key 的queue
    - topic：通配符，message 转至符合 routing pattern 的 queue
  - exchange 只转发 message，不存 message
    - exchange 没有 queue 与之绑定时，丢失消息
    - 没有 queue 符合 message 的路由规则时，丢失消息
- 应用场景：
  - 发送 支付成功 的 message 到 exchange
  - 下游服务均注册 queue 监听该 message 做业务操作

#### routing

![](https://www.rabbitmq.com/img/tutorials/python-four.png)

- 路由模式
  - 要求 queue 绑定 exchange 时，必须指定 routing key
  - message 会转发至符合 routing key 的 queue 中

#### topics

![](https://www.rabbitmq.com/img/tutorials/python-five.png)

- 主题模式
  - 支持通配符
    - `#` 匹配一个或多个词
    - `*` 只能匹配一个词

#### rpc

![](https://www.rabbitmq.com/img/tutorials/python-six.png)



