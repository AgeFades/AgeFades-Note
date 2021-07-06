[TOC]

# 慕课 - RabbitMq

##主流消息中间件介绍

### MQ 衡量指标

```shell
# 服务性能

# 数据存储

# 集群架构
```

### AcitiveMQ

```shell
# ActiveMQ 是 Apache 出品，最流行的，能力强劲的开源消息总线。

# 完全支持 JMS 规范的消息中间件。
```

#### Master-Slave 模式图解:

![UTOOLS1577955632489.png](http://yanxuan.nosdn.127.net/11f744d7df5c69883819c493467e9a46.png)

```shell
# 利用 Zookeeper 作为协调节点

# 平时主节点 Master 对外提供服务

# 当 Master 宕机时，Zookeeper 将某个 Slave 切换成 Master 继续提供服务
```

#### NetWork 模式图解:

![UTOOLS1577955725304.png](http://yanxuan.nosdn.127.net/20db65f25f3addf3610df627869f3aff.png)

```shell
# NetWork 本质上是多组 Master-Slave 模式的组成

# 中间利用 network 网关通信，达到分布式集群的目的
```

### Kafka

```shell
# Kafka 是 Linkedln 开源的分布式 发布-订阅 消息系统

# 目前归属于 Apache 顶级项目。

# 主要特点:
	# 基于 Pull 的模式来处理消息消费，追求高吞吐量
	# 一开始的目的就是用于日志收集和传输
	# 0.8 版本开始支持复制，不支持事务
	# 对消息的重复、丢失、错误没有严格要求
	# 适合产生大量数据的互联网服务的数据收集业务
```

#### Kafka 集群模式图解:

![UTOOLS1577956149885.png](http://yanxuan.nosdn.127.net/064d18626d25d85c61722b2098b8655f.png)

### RocketMQ

```shell
# RocketMQ 是阿里开源的消息中间件
	# 目前也已经孵化为 Apache 顶级项目
	# 纯 Java 开发

# 具有 高吞吐量、高可用性、适合大规模分布式系统应用 的特点

# RocketMQ 思路起源于 Kafka
	# 对消息的可靠性传输及事务性做了优化
	# 目前在阿里集团被广泛应用于 交易、充值、流计算、消息推送、日志流式处理、binlog 分发等场景
```

#### 集群拓扑图解:

![UTOOLS1577956535744.png](http://yanxuan.nosdn.127.net/3dfd03a808fd181589840a9093726601.png)

### RabbitMQ

```shell
# RabbitMQ 是使用 Erlang 语言开发的开源消息队列系统

# 基于 AMQP 协议来实现

# AMQP 的主要特征:
	# 面向消息、队列、路由（包括 点对点 和 发布/订阅 ）、可靠性、安全

# AMQP 协议更多用在企业系统内
	# 对数据一致性、稳定性和可靠性要求很高的场景
	# 对性能吞和吞吐量的要求还在其次
```

#### RabbitMQ 集群图解

![UTOOLS1577956918239.png](http://yanxuan.nosdn.127.net/392ea14cf89d879ab00a3c4dceb4c60e.png)

## RabbitMQ 核心概念及 AMQP协议

### 初始 RabbitMQ

```shell
# RabbitMQ 是一个开源的消息代理和队列服务器

# 用来通过 普通协议 在完全不同的应用之间共享数据

# RabbitMQ  是使用 Erlang 语言来编写的，并基于 AMQP 协议
```

### 哪些大厂在用 RabbitMQ，为什么？

```shell
# 滴滴、美团、头条、去哪儿、艺龙...

# 原因:
	# 开源、性能优秀、稳定性保障
	# 提供可靠性消息投递模式（confirm）、返回模式（return）
	# 与 SpringAMQP 完美的整合、API 丰富
	# 集群模式丰富、表达式配置、HA 模式、镜像队列模型
	# 保证数据不丢失的前提做到高可靠性、可用性
```

### RabbitMQ  高性能的原因？

```shell
# Erlang 语言最初用于交换机领域的架构模式
	# 这使得 RabbitMQ 在 Broker 之间进行数据交互的性能是非常优秀的
	
# Erlang 的优点:
	# Erlang 有着和原生 Socket 一样的延迟
```

### 什么是 AMQP 高级消息队列协议？

```shell
# AMQP 全称: Advanced Message Queuing protocol
	# 翻译过来就是: 高级消息队列协议
	
# AMQP 定义: 是具有现代特征的二进制协议
	# 是一个提供统一消息服务的应用层标准高级消息队列协议
	# 是应用层协议的一个开放标准
	# 为面向消息的中间件设计
```

#### AMQP 协议模型图解:

![UTOOLS1577958282128.png](http://yanxuan.nosdn.127.net/d09a445f480888be4d14320675560db1.png)

### AMPQ 核心概念

#### Server

```shell
# 又称 Broker，接受客户端的连接，实现 AMQP 实体服务
```

#### Connection

```shell
# 连接，应用程序与 Broker 的网络连接
```

#### Channel

```shell
# 网络信道，几乎所有的操作都在 Channel 中进行

# Channel 是进行消息读写的通道

# 客户端可建立多个 Channel

# 每个 Channel 代表一个会话任务
```

#### Message

```shell
# 消息，服务器和应用程序之间传送的数据

# 由 Properties 和 Body 组成

# Properties 可以对消息进行修饰
	# 比如消息的优先级、延迟 等高级特性
	
# Body 就是消息体内容

# 常用属性:
	# delivery mode : 送达模式，可以选择持久化或非持久化
	# headers : 自定义属性
	# content_type : 消息类型
	# content_encoding : 编码类型
	# priority : 优先级
	
# 其他属性:
	# correlation_id : 消息的唯一Id（常用于消息的路由、ACK、幂等...）
	# reply_to : 消息失败后返回哪个队列
	# expiration : 消息的过期时间
	# message_id、timestamp、type、user_id、app_id、cluster_id...
```

#### Virtual host

```shell
# 虚拟地址，用于进行逻辑隔离，最上层的消息路由

# 一个 Virtual Host 里面可以有若干个 Exchange 和 Queue

# 同一个 Virtual Host 里面不能有相同名称的 Exchange 或 Queue
```

#### Exchange

```shell
# 交换机，接收信息，根据路右键转发消息到绑定的队列
```

#### Binding

```shell
# Exchange 和 Queue 之间的虚拟连接，binding 中可以包含 routing key
```

#### Routing key

```shell
# 一个路由规则，虚拟机可用它来确定如何路由一个特定消息
```

#### Queue

```shell
# 也称为 Message Queue，消息队列，保存消息并将它们转发给消费者
```

### RabbitMQ 整体架构图解

![UTOOLS1578015453053.png](http://yanxuan.nosdn.127.net/a6bc61531b257a282eae37f43beb1d06.png)

#### RabbitMQ 消息流转图解

![UTOOLS1578015575619.png](http://yanxuan.nosdn.127.net/44e277b3e7e32d54b4d17b843ceecf0a.png)

## RabbitMQ 安装与使用

```shell
# 官网地址: http://www.rabbitmq.com

# 此处本人使用 docker 安装，具体命令请参考 运维 -> Linux 基础开发环境搭建.md
```

## 命令行与管控台

### 命令行

```shell
## 以下命令皆可以用 docker exec -it rabbit bash 尝试操作 
# 启动应用
rabbitmqctl start_app

# 关闭应用
rabbitmqctl stop_app

# 节点状态
rabbitmqctl status

# 添加用户
rabbitmqctl add_user username password

# 列出所有用户
rabbitmqctl list_users

# 删除用户
rabbitmqctl delete_user username

# 清除用户权限
rabbitmqctl clear_permissions -p vhostpath username

# 列出用户权限
rabbitmqctl list_user_permissions username

# 修改密码
rabbitmqctl change_password username newpassword

# 赋予用户超级管理员权限
rabbitmqctl set_user_tags username administrator

# 设置用户虚拟主机权限
rabbitmqctl set_permissions -p vhostpath username ".*" ".*" ".*" 

# 创建虚拟主机
rabbitmqctl add_vhost vhostpath

# 列出所有虚拟主机
rabbitmqctl list_vhosts

# 列出虚拟主机上所有权限
rabbitmqctl list_permissions -p vhostpath

# 删除虚拟主机
rabbitmqctl delete_vhost vhostpath

# 查看所有队列消息
rabbitmqctl list_queues

# 清除队列里的消息
rabbitmqctl -p vhostpath purge_queue blue

# 移除所有数据，要在 rabbitmqctl stop_app 之后使用
rabbitmqctl reset

# 组成集群命令
rabbitmqctl join_cluster <clusterNode> [--ram]

# 查看集群状态
rabbitmqctl cluster_status

# 修改集群节点的存储形式
rabbitmqctl change_cluster_node_type disc | ram

# 摘除节点
rabbitmqctl forget_cluster_node [--offline]

# 修改节点名称
rabbitmqctl rename_cluster_node oldNode1 newNode1 [oldNode2] [newNoed2]...
```

### 控制台

```shell
# 左上角为 RabbitMQ 的 Logo、版本、Erlang 语言的版本
```

![UTOOLS1578032835191.png](http://yanxuan.nosdn.127.net/e5bebccdbdc95fdd8bb204344dd9dd69.png)

```shell
# 右上角依次为:
	# 数据刷新时间间隔
	# 当前使用的虚拟主机
	# 节点主机名
	# 当前用户以及退出登录操作
```

![UTOOLS1578030621678.png](http://yanxuan.nosdn.127.net/048ba58632da755dd247e58f365fd74f.png)

```shell
# 标签页依次为:
	# Overview : 主览
	# Connections : 连接
	# Channels : 网络通道
	# Exchanges : 交换机
	# Queues : 队列
	# Admin : 对用户、虚拟主机、权限的操作
```

![UTOOLS1578032797886.png](http://yanxuan.nosdn.127.net/34f8b9db6f4ed6c7c4bb247010c22eb5.png)

## 急速入门 - 消息生产与消费

### ConncetionFactory

```shell
# 获取连接工厂
	# 配置节点的 Virtual host、ip、port
```

### Connection

```shell
# 一个连接
```

### Channel

```shell
# 数据通信信道，可发送和接收消息
```

### Queue

```shell
# 具体的消息存储队列
```

### Producer & Consumer

```shell
# 生产和消费者
```

### Java 代码

#### Producer 消息生产者

```java
public class Producer {

    public static void main(String[] args) throws Exception {
        // 1. 创建一个ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 2. 创建连接工厂创建连接
        Connection connection = factory.newConnection();

        // 3. 通过 connection 创建一个 Channel
        Channel channel = connection.createChannel();

        // 4. 通过 Channel 发送数据
        String body = "Hello RabbitMQ!";
        for (int i = 0; i < 5; i++) {
            channel.basicPublish("","test001", null, body.getBytes());
        }

        // 5. 关闭相关连接
        channel.close();
        connection.close();
    }

}
```

```shell
# 生产者主要方法: 发送一条消息
channel.basicPublish()
```

```java
/**
     * Publish a message.
     *
     * Publishing to a non-existent exchange will result in a channel-level
     * protocol exception, which closes the channel.
     *
     * Invocations of <code>Channel#basicPublish</code> will eventually block if a
     * <a href="http://www.rabbitmq.com/alarms.html">resource-driven alarm</a> is in effect.
     *
     * @see com.rabbitmq.client.AMQP.Basic.Publish
     * @see <a href="http://www.rabbitmq.com/alarms.html">Resource-driven alarms</a>
     * @param exchange the exchange to publish the message to
     * @param routingKey the routing key
     * @param props other properties for the message - routing headers etc
     * @param body the message body
     * @throws java.io.IOException if an error is encountered
     */
    void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;
```

```shell
# 源码参数解释:
	# exchange : 要将消息发布到的交换器
	# routingKey : 队列路由键
	# props : 消息路由头等的其他属性
	# body : 消息体的 byte 数组
```

#### Consumer 消息消费者

```java
public class Consumer {

    public static void main(String[] args) throws Exception {
        // 1. 创建一个ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 2. 创建连接工厂创建连接
        Connection connection = factory.newConnection();

        // 3. 通过 connection 创建一个 Channel
        Channel channel = connection.createChannel();

        // 4. 声明一个队列
        String queueName = "test001";
        channel.queueDeclare(queueName, true, false, false, null);

        // 5. 创建消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);

        // 6. 设置 Channel
        channel.basicConsume(queueName, true, consumer);

        // 7. 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String msg = new String(delivery.getBody());
            System.out.println("消费端收到消息: " + msg);
            Envelope envelope = delivery.getEnvelope();
            // 消息的唯一标签，确保消息的不重复消费
            long deliveryTag = envelope.getDeliveryTag();
            System.out.println("deliveryTag: " + deliveryTag);
        }

    }

}
```

```shell
# 消费者主要方法 : 声明队列
channel.queueDeclare()
```

```java
 /**
     * Declare a queue
     * @see com.rabbitmq.client.AMQP.Queue.Declare
     * @see com.rabbitmq.client.AMQP.Queue.DeclareOk
     * @param queue the name of the queue
     * @param durable true if we are declaring a durable queue (the queue will survive a server restart)
     * @param exclusive true if we are declaring an exclusive queue (restricted to this connection)
     * @param autoDelete true if we are declaring an autodelete queue (server will delete it when no longer in use)
     * @param arguments other properties (construction arguments) for the queue
     * @return a declaration-confirm method to indicate the queue was successfully declared
     * @throws java.io.IOException if an error is encountered
     */
    Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                                 Map<String, Object> arguments) throws IOException;
```

```shell
# 源码参数解释:
	# queue : 队列名称
	# durable : 队列是否持久化
	# exclusive : 是否独占，是的话就只能被一个连接使用（可以用来确保消息的顺序性投递）
	# autoDelete : 如果没有交换机与之绑定，是否自动删除
	# arguments : 扩展参数
```

```shell
# 消费者主要方法 : 设置 Channel
channel.basicConsumer()
```

```java
/**
     * Start a non-nolocal, non-exclusive consumer, with
     * a server-generated consumerTag.
     * @param queue the name of the queue
     * @param autoAck true if the server should consider messages
     * acknowledged once delivered; false if the server should expect
     * explicit acknowledgements
     * @param callback an interface to the consumer object
     * @return the consumerTag generated by the server
     * @throws java.io.IOException if an error is encountered
     * @see com.rabbitmq.client.AMQP.Basic.Consume
     * @see com.rabbitmq.client.AMQP.Basic.ConsumeOk
     * @see #basicConsume(String, boolean, String, boolean, boolean, Map, Consumer)
     */
    String basicConsume(String queue, boolean autoAck, Consumer callback) throws IOException;
```

```shell
# 源码参数解释:
	# queue : 队列名称
	# autoAck : 消费端是否自动 ack，默认为 true，设置为 false 可以用来确保消息百分百投递的机制
	# callback : 具体的消费对象
```

### 程序执行结果

```shell
# 在程序运行时，可以观察到 Web 管理端的 Connection、Channel 等连接信息发生变化...
```

![UTOOLS1578037860495.png](http://yanxuan.nosdn.127.net/3226906605639d166c43434ae87cbeba.png)

```shell
# 在这里我们对 Exchange 给定的值是空
	# 默认就会归为 AMQP default 这个交换机
	# 这个交换机的路由规则就必须是有相同名字的队列才能消费
```

## Exchange 交换机

### 概念

```shell
# Exchange : 接收消息，并根据路由键转发消息所绑定的队列
```

### 图解

![UTOOLS1578039184063.png](https://i.loli.net/2020/01/03/aUfQHck16DszTdB.png)

### 属性

```shell
# Name : 交换机名称

# Type : 交换机类型 direct、topic、fanout、headers

# Durability : 是否需要持久化，true 为持久化

# Auto Delete : 当最后一个绑定到 Exchange 上的队列删除后，自动删除该 Exchange

# Internal : 当前 Exchange 是否用于 RabbitMQ 内部使用，默认为 False	

# Arguments : 扩展参数，用于扩展 AMQP 协议定制化使用
```

### Direct Exchange

```shell
# 所有发送到 Direct Exchange 的消息被转发到 RouteKey 中指定的 Queue

# 注意:
	# Direct 模式可以使用 RabbitMQ 自带的 Exchange: default Exchange
	# 所以不需要将 Exchange 进行任何绑定（binding）操作
	# RouteKey 必须完全匹配才会被队列接收，否则该消息会被抛弃
```

#### 图解

![l69CsP.png](https://s2.ax1x.com/2020/01/07/l69CsP.png)

#### 代码

```java
public class Producer4DirectExchange {

    public static void main(String[] args) throws Exception{

        // 1. 创建连接工厂
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 2. 创建 Connection
        Connection connection = factory.newConnection();

        // 3. 创建 Channel
        Channel channel = connection.createChannel();

        // 4. 声明
        String exchangeName = "amq.direct";
        String routingKey = "test.direct";

        // 5. 发送
        String msg = "Hello World RabbitMQ 4 Direct Exchange Message 111 ...";
        channel.basicPublish(exchangeName, routingKey, null, msg.getBytes());

        // 6. 关闭相关连接
        channel.close();
        connection.close();
    }

}
```

```java
public class Consumer4DirectExchange {

    public static void main(String[] args) throws Exception{

        ConnectionFactory factory = new ConnectionFactory() ;
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        factory.setAutomaticRecoveryEnabled(true);
        factory.setNetworkRecoveryInterval(3000);

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        // 声明
        String exchangeName = "amq.direct";
        String exchangeType = "direct";
        String queueName = "test_direct_queue";
        String routingKey = "test.direct";

        // 声明交换机
        channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
        // 声明队列
        channel.queueDeclare(queueName, false, false, false, null);
        // 建立绑定关系
        channel.queueBind(queueName, exchangeName, routingKey);

        QueueingConsumer consumer = new QueueingConsumer(channel);
        channel.basicConsume(queueName, true, consumer);
        // 循环获取消息
        while (true) {
            // 获取消息，如果没有消息，这一步将会一直阻塞
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String msg = new String(delivery.getBody());
            System.out.println("收到消息: " + msg);
        }

    }

}
```

### Topic Exchange

```shell
# 所有发送到 Topic Exchange 的消息被转发到所有关心 RoutingKey 中指定 Topic 的 Queue 上

# Exchange 将 RoutingKey 和某 Topic 进行模糊匹配，此时队列需要绑定一个 Topic

# 注意: 可以使用通配符进行模糊匹配
	# 符号 "#" 匹配一个或多个词
	# 符号 "*" 匹配不多不少一个词
	# 例如:
		# "log.#" 能够匹配到 "log.info.oa"
		# "log.*" 只会匹配到 "log.error"
```

#### 图解

![UTOOLS1578363322074.png](http://yanxuan.nosdn.127.net/cf6ea53bf54f3e8dc5cec785e0307846.png)

#### 代码

```java
public class Producer4TopicExchange {

    public static void main(String[] args) throws Exception{

        // 1. 创建ConnectionFactory
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");
        // 2. 创建Connection
        Connection connection = connectionFactory.newConnection();
        // 3. 创建Channel
        Channel channel = connection.createChannel();

        // 4. 声明
        String exchangeName = "test_topic_exchange";
        String routingKey1 = "user.save";
        String routingKey2 = "user.update";
        String routingKey3 = "user.delete.abc";

        // 5. 发送
        String msg = "Hello World RabbitMQ 4 Topic Exchange Message...";
        channel.basicPublish(exchangeName, routingKey1, null, msg.getBytes());
        channel.basicPublish(exchangeName, routingKey2, null, msg.getBytes());
        channel.basicPublish(exchangeName, routingKey3, null, msg.getBytes());
        channel.close();
        connection.close();

    }

}
```

```java
public class Consumer4TopicExchange {

    public static void main(String[] args) throws Exception{

        ConnectionFactory connectionFactory = new ConnectionFactory() ;

        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");

        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();

        Channel channel = connection.createChannel();

        // 4. 声明
        String exchangeName = "test_topic_exchange";
        String exchangeType = "topic";
        String queueName = "test_topic_queue";
        String routingKey = "user.#";

        channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
        channel.queueDeclare(queueName, false, false, false, null);
        channel.queueBind(queueName, exchangeName, routingKey);

        QueueingConsumer consumer = new QueueingConsumer(channel);
        channel.basicConsume(queueName, true, consumer);
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String msg = new String(delivery.getBody());
            System.out.println("收到消息：" + msg);
        }

    }

}
```

### Fanout Exchange

```shell
# 不处理路由键，只需要简单的将队列绑定到交换机上

# 发送到交换机的消息都会被转发到与该交换机绑定的所有队列上

# Fanout 交换机转发消息是最快的
```

#### 图解

![UTOOLS1578364676912.png](http://yanxuan.nosdn.127.net/20dbe66537090c5a84cea709b8ae3198.png)

#### 代码

```java
public class Producer4FanoutExchange {

	
	public static void main(String[] args) throws Exception {
		
		//1 创建ConnectionFactory
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		//2 创建Connection
		Connection connection = connectionFactory.newConnection();
		//3 创建Channel
		Channel channel = connection.createChannel();  
		//4 声明
		String exchangeName = "test_fanout_exchange";
		//5 发送
		for(int i = 0; i < 10; i ++) {
			String msg = "Hello World RabbitMQ 4 FANOUT Exchange Message ...";
			channel.basicPublish(exchangeName, "", null , msg.getBytes()); 			
		}
		channel.close();  
        connection.close();  
	}
	
}
```

```java
public class Consumer4FanoutExchange {

	public static void main(String[] args) throws Exception {
		
        ConnectionFactory connectionFactory = new ConnectionFactory() ;  
        
        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
        connectionFactory.setAutomaticRecoveryEnabled(true);
        connectionFactory.setNetworkRecoveryInterval(3000);
        Connection connection = connectionFactory.newConnection();
        
        Channel channel = connection.createChannel();  
		//4 声明
		String exchangeName = "test_fanout_exchange";
		String exchangeType = "fanout";
		String queueName = "test_fanout_queue";
		String routingKey = "";	//不设置路由键
		channel.exchangeDeclare(exchangeName, exchangeType, true, false, false, null);
		channel.queueDeclare(queueName, false, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
        //durable 是否持久化消息
        QueueingConsumer consumer = new QueueingConsumer(channel);
        //参数：队列名称、是否自动ACK、Consumer
        channel.basicConsume(queueName, true, consumer); 
        //循环获取消息  
        while(true){  
            //获取消息，如果没有消息，这一步将会一直阻塞  
            Delivery delivery = consumer.nextDelivery();  
            String msg = new String(delivery.getBody());    
            System.out.println("收到消息：" + msg);  
        } 
		}
}
```

## Message 属性

```java
public class Producer {

    public static void main(String[] args) throws Exception {
        // 1. 创建一个ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 2. 创建连接工厂创建连接
        Connection connection = factory.newConnection();

        // 3. 通过 connection 创建一个 Channel
        Channel channel = connection.createChannel();

        HashMap<String, Object> map = new HashMap<>();
        map.put("id", 1);
        map.put("name", "tony");

        AMQP.BasicProperties properties = new AMQP.BasicProperties().builder()
                .deliveryMode(2) // 2为持久化消息传递，服务重启后还在，1则相反
                .contentEncoding("UTF-8")
                .expiration("10000") // 过期时间
                .headers(map)   // 自定义属性
                .build();

        // 4. 通过 Channel 发送数据
        String body = "Hello RabbitMQ!";
        for (int i = 0; i < 5; i++) {
            channel.basicPublish("","test001", properties, body.getBytes());
        }

        // 5. 关闭相关连接
        channel.close();
        connection.close();
    }

}
```

```java
public class Consumer {

    public static void main(String[] args) throws Exception {
        // 1. 创建一个ConnectionFactory
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("127.0.0.1");
        factory.setPort(5672);
        factory.setVirtualHost("/");

        // 2. 创建连接工厂创建连接
        Connection connection = factory.newConnection();

        // 3. 通过 connection 创建一个 Channel
        Channel channel = connection.createChannel();

        // 4. 声明一个队列
        String queueName = "test001";
        channel.queueDeclare(queueName, true, false, false, null);

        // 5. 创建消费者
        QueueingConsumer consumer = new QueueingConsumer(channel);

        // 6. 设置 Channel
        channel.basicConsume(queueName, true, consumer);

        // 7. 获取消息
        while (true) {
            QueueingConsumer.Delivery delivery = consumer.nextDelivery();
            String msg = new String(delivery.getBody());
            System.out.println("消费端收到消息: " + msg);
            Envelope envelope = delivery.getEnvelope();
            // 消息的唯一标签，确保消息的不重复消费
            long deliveryTag = envelope.getDeliveryTag();
            System.out.println("deliveryTag: " + deliveryTag);

            AMQP.BasicProperties properties = delivery.getProperties();
            System.out.println(properties.getContentEncoding());
           	Map<String, Object> headers = properties.getHeaders();
            System.out.println(headers.get("id"));
        }

    }
	
}
```

## RabbitMQ 高级特性

### 消息如何保障 100% 的投递成功？

#### 什么是生产端的可靠性传递

```shell
# 保障消息的成功发出

# 保障 MQ 节点的成功接收

# 发送端收到 MQ 节点（Borker）确认应答

# 完善的消息进行补偿机制
```

#### 业界主流的解决方案

```shell
# 消息落库，对消息状态进行打标

# 消息的延迟投递，做二次确认，回调检查
```

##### 消息落库方案图解

![UTOOLS1578453925039.png](http://yanxuan.nosdn.127.net/3cbd31c8a40854982da50d654c77e1f2.png)

```shell
# 保障 MQ 我们思考如果第一种可靠性传递，在高并发的场景下是否合适？
	# 入库 IO 次数过多，影响性能
```

##### 消息的延迟投递，做二次确认，回调检查

![UTOOLS1578463771652.png](http://yanxuan.nosdn.127.net/d40d92070fe2f818a025efccbacde984.png)

### 幂等性

#### 概念

```shell
# 幂等性是什么?
	# 可以借鉴数据库的乐观锁机制
	# 比如执行一条更新库存的 SQL 语句
	# UPDATE T SET COUNT = COUNT - 1,VERSION = VERSION + 1 WHERE VERSION = 1
```

#### 消费端-幂等性保障

```shell
# 在海量订单产生的业务高峰期，如何避免消息的重复消费问题？

# 消费端实现幂等性，就意味着，我们的消息永远不会消费多次
	# 即使我们收到了多条一样的消息
```

#### 业界主流的幂等性操作

```shell
# 唯一ID + 指纹码 机制，利用数据库主键去重

# 利用 Redis 的原子性去实现
```

### Confirm 确认消息

#### 理解 Confirm 消息确认机制

```shell
# 消息的确认，是指生产者投递消息后，如果 Broker 收到消息，则会给我们生产者一个应答

# 生产者接收应答，用来确定这条消息是否正常的发送到 Broker

# 这种方式也是消息的可靠性投递的核心保障
```

#### 确认机制流程图

![UTOOLS1578465418042.png](http://yanxuan.nosdn.127.net/6e47f0b53c9d5560f3d4abe79b470bda.png)

#### 如何实现 Confirm 确认消息

```shell
# 第一步: 在 channel 上开启确认模式:
channel.confirmSelect()

# 第二步: 在 channel 上添加监听:
# 监听成功和失败的返回结果，根据具体的结果对消息进行重新发送、记录日志等后续处理
addConfirmListener
```

#### 代码

```java
public class Producer {

    public static void main(String[] args) throws Exception {

        //1 创建ConnectionFactory
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("127.0.0.1");
        connectionFactory.setPort(5672);
        connectionFactory.setVirtualHost("/");

        //2 获取Connection
        Connection connection = connectionFactory.newConnection();

        //3 通过Connection创建一个新的Channel
        Channel channel = connection.createChannel();

        //4 指定我们的消息投递模式: 消息的确认模式
        channel.confirmSelect();

        String exchangeName = "test_confirm_exchange";
        String routingKey = "confirm.save";

        //5 发送一条消息
        String msg = "Hello RabbitMQ Send confirm message!";
        for (int i = 0; i < 10; i++) {
            channel.basicPublish(exchangeName, routingKey, null, msg.getBytes());
        }

        //6 添加一个确认监听
        channel.addConfirmListener(new ConfirmListener() {
            @Override
            public void handleNack(long deliveryTag, boolean multiple) throws IOException {
                System.err.println("deliveryTag no ack! :" + deliveryTag );
            }

            @Override
            public void handleAck(long deliveryTag, boolean multiple) throws IOException {
                System.err.println("deliveryTag ack! :" + deliveryTag );
            }
        });

    }
}
```

```java
public class Consumer {
	
	public static void main(String[] args) throws Exception {

		//1 创建ConnectionFactory
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		//2 获取Connection
		Connection connection = connectionFactory.newConnection();
		
		//3 通过Connection创建一个新的Channel
		Channel channel = connection.createChannel();
		
		String exchangeName = "test_confirm_exchange";
		String routingKey = "confirm.#";
		String queueName = "test_confirm_queue";
		
		//4 声明交换机和队列 然后进行绑定设置, 最后制定路由Key
		channel.exchangeDeclare(exchangeName, "topic", true);
		channel.queueDeclare(queueName, true, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		//5 创建消费者 
		QueueingConsumer queueingConsumer = new QueueingConsumer(channel);
		channel.basicConsume(queueName, false, queueingConsumer);
		
		while(true){
			Delivery delivery = queueingConsumer.nextDelivery();
			String msg = new String(delivery.getBody());
			long deliveryTag = delivery.getEnvelope().getDeliveryTag();
			System.err.println("消费端: " + msg + "deliveryTag: " + deliveryTag);
			channel.basicAck(deliveryTag, false);
		}

	}
}
```

### Return 消息机制

```shell
# Return Listener 用于处理一些不可路由的消息

# 消息生产者，通过指定一个 Exchange 和 RoutingKey
	# 把消息送达到某一个队列中去
	# 然后消费者监听队列，进行消费处理操作
	
# 但是在某些情况下， 如果我们在发送消息的时候
	# 当前的 exchange 不存在或者指定的路由 key 路由不到
	# 这时候如果需要监听这种不可达的消息，就要使用 Return Listener
	
# 在基础 API 中有一个关键的配置项: Mandatory
	# 如果为 true，则监听器会接收到路由不可达的消息
	# 然后进行后续处理，如果为 false，那么 broker 端自动删除该消息
```

#### 流程图解

![UTOOLS1578536089869.png](http://yanxuan.nosdn.127.net/2cd672cec4cd53ab5b2bec206ea6ae54.png)

#### 代码

```java
public class Producer {
	
	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchange = "test_return_exchange";
		String routingKey = "return.save";
		String routingKeyError = "abc.save";
		
		String msg = "Hello RabbitMQ Return Message";
		
		channel.addReturnListener(new ReturnListener() {
			@Override
			public void handleReturn(int replyCode, String replyText, String exchange,
					String routingKey, BasicProperties properties, byte[] body) throws IOException {
				
				System.err.println("---------handle  return----------");
				System.err.println("replyCode: " + replyCode);
				System.err.println("replyText: " + replyText);
				System.err.println("exchange: " + exchange);
				System.err.println("routingKey: " + routingKey);
				System.err.println("properties: " + properties);
				System.err.println("body: " + new String(body));
			}
		});
		
		channel.basicPublish(exchange, routingKeyError, true, null, msg.getBytes());
		
		//channel.basicPublish(exchange, routingKey, true, null, msg.getBytes());
		
	}
}
```

```java
public class Consumer {
	
	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchangeName = "test_return_exchange";
		String routingKey = "return.#";
		String queueName = "test_return_queue";
		
		channel.exchangeDeclare(exchangeName, "topic", true, false, null);
		channel.queueDeclare(queueName, true, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		QueueingConsumer queueingConsumer = new QueueingConsumer(channel);
		
		channel.basicConsume(queueName, true, queueingConsumer);
		
		while(true){
			
			Delivery delivery = queueingConsumer.nextDelivery();
			String msg = new String(delivery.getBody());
			System.err.println("消费者: " + msg);
		}
		
	}
}
```

### 消费端自定义监听

```shell
# 前面一般都是在代码中编写 while 循环
	# 进行 consumer.nextDelivery 方法进行获取下一条消息
	# 然后进行消费处理
	
# 而实际工作中最常用的使用方式是:
	# 自定义的 Consumer，更加方便，解耦性也更强
```

#### 代码

```java
public class Producer {

	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchange = "test_consumer_exchange";
		String routingKey = "consumer.save";
		
		String msg = "Hello RabbitMQ Consumer Message";
		
		for(int i =0; i<5; i ++){
			channel.basicPublish(exchange, routingKey, true, null, msg.getBytes());
		}
		
	}
}
```

```java
public class Consumer {

	public static void main(String[] args) throws Exception {

		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();

		String exchangeName = "test_consumer_exchange";
		String routingKey = "consumer.#";
		String queueName = "test_consumer_queue";
		
		channel.exchangeDeclare(exchangeName, "topic", true, false, null);
		channel.queueDeclare(queueName, true, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		channel.basicConsume(queueName, true, new MyConsumer(channel));

	}
}
```

```java
public class MyConsumer extends DefaultConsumer {

	public MyConsumer(Channel channel) {
		super(channel);
	}

	@Override
	public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		System.err.println("-----------consume message----------");
		System.err.println("consumerTag: " + consumerTag);
		System.err.println("envelope: " + envelope);
		System.err.println("properties: " + properties);
		System.err.println("body: " + new String(body));
	}

}
```

### 消费端限流

```shell
# 什么是消费端的限流？

# 假设一个场景:
	# RabbitMQ 服务器上有上万条未处理的消息
	# 随便打开一个消费者客户端，会出现下面情况
	# 巨量的消息瞬间全部推送过来，但是单个客户端无法同时处理这么多数据
	
# RabbitMQ 提供了一种 qos（服务质量保证）功能
	# 即在非自动确认消息的前提下
	# 如果一定数目的消息（通过基于 consumer 或者 channel 设置 Qos 的值）
	# 未被确认前，不进行消费新的消息
	
# 核心方法:
# prefetchSize: 0 消息的大小限制
# prefetchCount: 消费者单次消费最大消息条数，没有ack完，消费者阻塞不会接受新的消息
# global: true/false 是否将上面设置应用于 channel，默认 false 为 consumer 级别
void basicQos(int prefetchSize, int prefetchCount, boolean global);

# prefetchSize 和 global 这两项，rabbitmq 没有实现
	# 暂且不研究 prefetch_count 和 no_ack=false 的情况下生效
	# 即在自动应答的情况下这两个值是不生效的
```

#### 代码

```java
public class Producer {

	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchange = "test_qos_exchange";
		String routingKey = "qos.save";
		
		String msg = "Hello RabbitMQ QOS Message";
		
		for(int i =0; i<5; i ++){
			channel.basicPublish(exchange, routingKey, true, null, msg.getBytes());
		}

	}
}
```

```java
public class Consumer {

	public static void main(String[] args) throws Exception {

		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();

		String exchangeName = "test_qos_exchange";
		String queueName = "test_qos_queue";
		String routingKey = "qos.#";
		
		channel.exchangeDeclare(exchangeName, "topic", true, false, null);
		channel.queueDeclare(queueName, true, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		//1 限流方式  第一件事就是 autoAck设置为 false
		channel.basicQos(0, 1, false);
		
		channel.basicConsume(queueName, false, new MyConsumer(channel));
	}
}
```

```java
public class MyConsumer extends DefaultConsumer {

	private Channel channel ;
	
	public MyConsumer(Channel channel) {
		super(channel);
		this.channel = channel;
	}

	@Override
	public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		System.err.println("-----------consume message----------");
		System.err.println("consumerTag: " + consumerTag);
		System.err.println("envelope: " + envelope);
		System.err.println("properties: " + properties);
		System.err.println("body: " + new String(body));

		try {
			TimeUnit.SECONDS.sleep(3);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}

		channel.basicAck(envelope.getDeliveryTag(), false);
		
	}

}
```

### 消费端 ACK 与重回队列

```shell
# 消费端的手工 ACK 和 NACK
	# 消费端进行消费的时候，如果由于业务异常我们可以进行日志的记录，然后进行补偿
	
	# 如果由于服务器宕机等严重问题，那就需要手工进行 ACK 保障消费端消费成功
	
# 消费端的重回队列
	# 消费端重回队列是为了对没有处理成功的消息，把消息重新投递给 broker
	
	# 在实际应用中，都会关闭重回队列，也就是设置为 false
```

#### 代码

```java
public class Producer {

	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchange = "test_ack_exchange";
		String routingKey = "ack.save";

		for(int i =0; i<5; i ++){
			
			Map<String, Object> headers = new HashMap<String, Object>();
			headers.put("num", i);
			
			AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
					.deliveryMode(2)
					.contentEncoding("UTF-8")
					.headers(headers)
					.build();
			String msg = "Hello RabbitMQ ACK Message " + i;
			channel.basicPublish(exchange, routingKey, true, properties, msg.getBytes());
		}
		
	}
}
```

```java
public class Consumer {

	public static void main(String[] args) throws Exception {

		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchangeName = "test_ack_exchange";
		String queueName = "test_ack_queue";
		String routingKey = "ack.#";
		
		channel.exchangeDeclare(exchangeName, "topic", true, false, null);
		channel.queueDeclare(queueName, true, false, false, null);
		channel.queueBind(queueName, exchangeName, routingKey);
		
		// 手工签收 必须要关闭 autoAck = false
		channel.basicConsume(queueName, false, new MyConsumer(channel));

	}
}
```

```java
public class MyConsumer extends DefaultConsumer {

	private Channel channel ;
	
	public MyConsumer(Channel channel) {
		super(channel);
		this.channel = channel;
	}

	@Override
	public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		System.err.println("-----------consume message----------");
		System.err.println("body: " + new String(body));
		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		if((Integer)properties.getHeaders().get("num") == 0) {
			channel.basicNack(envelope.getDeliveryTag(), false, true);
		} else {
			channel.basicAck(envelope.getDeliveryTag(), false);
		}
		
	}

}
```

### TTL 队列/消息

```shell
# TTL 是 Time To Live 的缩写，也就是生存时间

# RabbitMQ 支持消息的过期时间，在消息发送时可以进行指定

# RabbitMQ 支持队列的过期时间，从消息入队列开始计算，只要超过了队列的超时时间配置，消息自动清除
```

### 死信队列

```shell
# DLX: Dead-Letter-Exchange

# 利用 DLX，当消息在一个队列中变成 死信 之后
	# 它能被重新 publish 到另一个 Exchange，这个 Exchange 就是 DLX
	
# 消息变成死信有以下几种情况:
	# 消息被拒绝(basic.reject/basic.nack) 并且 requeue=false
	# 消息 TTL 过期
	# 队列达到最大长度
	
# DLX 也是一个正常的 Exchange，和一般的 Exchange 没有区别
	# 它能在任何的队列上被指定，实际上就是设置某个队列的属性
	
# 可以监听这个队列中消息做相应的处理，这个特性可以弥补 RabbitMQ3.0 以前支持的 immediate 参数的功能

# 死信队列的设置:
	# 首先需要设置死信队列的 exchange 和 queue，然后进行绑定:
		# Exchange: dlx.exchange
		# Queue: dlx.queue
		# RoutingKey: #
		
	# 然后进行正常声明交换机、队列、绑定
		# 需要在队列加上一个参数即可:
			# arguments.put("x-dead-letter-exchange", "dlx.exchange")
			
	# 这样消息在过期、requeue、队列在达到最大长度时，消息就可以直接路由到死信队列
```

#### 代码

```java
public class Producer {
	
	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		String exchange = "test_dlx_exchange";
		String routingKey = "dlx.save";
		
		String msg = "Hello RabbitMQ DLX Message";
		
		for(int i =0; i<1; i ++){
			
			AMQP.BasicProperties properties = new AMQP.BasicProperties.Builder()
					.deliveryMode(2)
					.contentEncoding("UTF-8")
					.expiration("10000")
					.build();
			channel.basicPublish(exchange, routingKey, true, properties, msg.getBytes());
		}
		
	}
}
```

```java
public class Consumer {
	
	public static void main(String[] args) throws Exception {
		
		ConnectionFactory connectionFactory = new ConnectionFactory();
		connectionFactory.setHost("127.0.0.1");
		connectionFactory.setPort(5672);
		connectionFactory.setVirtualHost("/");
		
		Connection connection = connectionFactory.newConnection();
		Channel channel = connection.createChannel();
		
		// 这就是一个普通的交换机 和 队列 以及路由
		String exchangeName = "test_dlx_exchange";
		String routingKey = "dlx.#";
		String queueName = "test_dlx_queue";
		
		channel.exchangeDeclare(exchangeName, "topic", true, false, null);
		
		Map<String, Object> agruments = new HashMap<String, Object>();
		agruments.put("x-dead-letter-exchange", "dlx.exchange");
		//这个agruments属性，要设置到声明队列上
		channel.queueDeclare(queueName, true, false, false, agruments);
		channel.queueBind(queueName, exchangeName, routingKey);
		channel.basicConsume(queueName, false, new MyConsumer(channel));
		
		//要进行死信队列的声明:
		channel.exchangeDeclare("dlx.exchange", "topic", true, false, null);
		channel.queueDeclare("dlx.queue", true, false, false, null);
		channel.queueBind("dlx.queue", "dlx.exchange", "#");
		QueueingConsumer queueingConsumer = new QueueingConsumer(channel);
		channel.basicConsume("dlx.queue", false, queueingConsumer);
		while(true){

			QueueingConsumer.Delivery delivery = queueingConsumer.nextDelivery();
			String msg = new String(delivery.getBody());
			System.err.println("死信队列消费消息: " + msg);
		}

	}
}
```

```java
public class MyConsumer extends DefaultConsumer {

	private Channel channel;

	public MyConsumer(Channel channel) {
		super(channel);
		this.channel = channel;
	}

	@Override
	public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
		System.err.println("-----------consume message----------");
		System.err.println("consumerTag: " + consumerTag);
		System.err.println("envelope: " + envelope);
		System.err.println("properties: " + properties);
		System.err.println("body: " + new String(body));
		channel.basicNack(envelope.getDeliveryTag(), false, false);
	}
	
}
```

## RabbitMQ 高级整合应用

### RabbitMQ 整合 Spring AMQP 实战

### RabbitAdmin

```shell
# RabbitAdmin 类可以很好的操作 RabbitMQ，在 Spring 中直接进行注入即可

# 注意: autoStartup 必须要设置为 true，否则 Spring 容器不会加载 RabbitAdmin 类

# RabbitAdmin 底层实现就是从 Spring 容器中获取
	# Exchange、Bingding、RoutingKey 以及 Queue 的 @Bean 声明
	
# 然后使用 RabbitTemplate 的 execute 方法执行对应的声明、修改、删除等一系列 RabbitMQ 基础功能操作
	# 例如: 添加一个交换机、删除一个绑定、清空一个队列里的消息等等
```

```java
@Configuration
@ComponentScan({"com.example.rabbitmq.*"})
public class RabbitMQConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        CachingConnectionFactory factory = new CachingConnectionFactory();
        factory.setAddresses("127.0.0.1:5672");
        factory.setUsername("guest");
        factory.setPassword("guest");
        factory.setVirtualHost("/");
        return factory;
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        return rabbitTemplate;
    }

    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory) {
        RabbitAdmin admin = new RabbitAdmin(connectionFactory);
        admin.setAutoStartup(true);
        return admin;
    }

}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RabbitmqApplicationTest {

    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    @Test
    public void testAdmin() throws Exception {
        rabbitAdmin.declareExchange(new DirectExchange("spring.test.direct3", false, false));
    }
  
  	@Test
    public void testBindingBuilder() throws Exception{
        rabbitAdmin.declareExchange(new DirectExchange("spring.test.topic.exchange", false, false));
        rabbitAdmin.declareQueue(new Queue("spring.test.topic.queue", false));
        rabbitAdmin.declareBinding(
                BindingBuilder
                        .bind(new Queue("spring.test.topic.queue", false))
                        .to(new TopicExchange("spring.test.topic.exchange", false, false))
                        .with("user.#")
        );
    }

}
```

### SpringAMQP 声明

```shell
# 在 Rabbit 基础API 里声明一个 Exchange、声明一个绑定、一个队列
channel.exchangeDeclare()
channel.queueDeclare()
channel.queueBind()

# 使用 SpringAMQP 去声明，即 @Bean 方式
```

```java
@Bean
public TopicExchange exchange() {
  return new TopicExchange("topic", true, false);
}

@Bean
public Queue queue() {
  return new Queue("queue", true);
}

@Bean
public Binding binding() {
  return BindingBuilder.bind(queue()).to(exchange()).with("spring.*");
}
```

```java
// 举例
/**
     * 针对消费者配置
     * 1. 设置交换机类型
     * 2. 将队列绑定到交换机
     *  FanoutExchange: 将消息发布到所有的绑定队列，无 RoutingKey 的概念
     *  HeadersExchange: 通过添加属性 Key-Value 匹配
     *  DirectExchange: 按照 RoutingKey 分发到指定队列
     *  TopicExchange: 多关键字匹配
     */
    @Bean
    public TopicExchange topicExchange() {
        return new TopicExchange("swordsman_topic_exchange", true, false);
    }

    @Bean
    public Queue topicSpringQueue() {
        return new Queue("topic_spring_queue", true);
    }

    @Bean
    public Binding topicSpringBinding() {
        return BindingBuilder.bind(topicSpringQueue()).to(topicExchange()).with("spring.*");
    }

    @Bean
    public Queue topicRabbitQueue() {
        return new Queue("topic_rabbit_queue", true);
    }

    @Bean
    public Binding topicRabbitBinding() {
        return BindingBuilder.bind(topicRabbitQueue()).to(topicExchange()).with("rabbit.*");
    }
```

### RabbitTemplate

```shell
# RabbitTemplate : 即消息模板

# 在于 SpringAMQP 整合的时候进行发送消息的关键类

# 该类提供了丰富的发送消息方法
	# 包括可靠性投递消息方法、回调监听消息接口 ConfirmCallback、
	# 返回值确认接口 ReturnCallback 等等..
	# 同样需要进行注入到 Spring 容器中，然后直接使用
	
# 在与 Spring 整合时需要实例化，但是在于 SpringBoot 整合时，在配置文件里添加配置即可
```

```java
		@Autowired
    private RabbitTemplate rabbitTemplate;

    @Test
    public void templateTest() {
        // 创建消息
        MessageProperties properties = new MessageProperties();
        properties.getHeaders().put("desc", "信息描述");
        properties.getHeaders().put("title", "标题");
        Message message = new Message("Hello RabbitMQ".getBytes(), properties);

        // 发送消息
        rabbitTemplate.convertAndSend("swordsman_topic_exchange",
                "spring.amqp",
                message,
                (msg) -> {
                    msg.getMessageProperties().getHeaders().put("ext", "额外消息");
                    return msg;
                });

    }
```

###  SimpleMessageListenerContainer

```shell
# 简单消息监听容器

# 这个类非常的强大，对于消费者的配置项，这个类都可以满足:
	# 监听队列（多个队列）、自动启动、自动声明功能

	# 设置事务特性、事务管理器、事务属性、事务容量（并发）、是否开启事务、回滚消息等

	# 设置消费者数量、最小最大数量、批量消费

	# 设置下消息确认和自动确认模式、是否重回队列、异常捕获 handler 函数
	
	# 设置消费者标签生成策略、是否独占模式、消费者属性等
	
	# 设置监听的监听器、消息转换器等等
	
# 注意:
	# SimpleMessageListenerContainer 可以进行动态设置
	# 比如在运行中的应用可以动态的修改其消费者数量的大小、接收消息的模式等
	# 很多基于 RabbitMQ 的自制定化后端管控台在进行动态设置的时候
		# 也是根据这一特性去实现的
		
# SimpleMessageListenerContainer 为什么可以动态感知配置变更？
```

```java
 @Bean
    public SimpleMessageListenerContainer container(ConnectionFactory connectionFactory) {
        SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
        container.setQueues(topicSpringQueue(), topicRabbitQueue());    // 监听的队列
        container.setConcurrentConsumers(1); // 当前消费者为1个
        container.setMaxConcurrentConsumers(5); // 最大消费者数量为5个
        container.setDefaultRequeueRejected(false); // 消息拒绝消费或消费失败 不重回队列
        container.setAcknowledgeMode(AcknowledgeMode.AUTO); // 设置签收 ACK 策略
        container.setConsumerTagStrategy(v -> v + "_" + UUID.randomUUID().toString()); // 设置消费端的标签名生成策略
        container.setMessageListener(msg -> System.out.println("消费者受到消息: " + new String(msg.getBody()) + "ConsumerTag: " + msg.getMessageProperties().getConsumerTag()));  // 设置消费端的消息监听器

        return container;
    }
```

###  MessageListenerAdapter

```shell
# 即消息监听适配器

# 核心属性:
	# defaultListenerMethod 默认监听方法名称: 用于设置监听方法名称
	# Delegate 委托对象: 实际真实的委托对象，用于处理消息
	# queueOrTagToMethodName: 队列标识与方法名称组成的集合
```

```java
public class MessageDelegate {

    public void handleMessage(String msg) {
        System.err.println("默认方法,消息内容为: " + msg);
    }

}
```

```java
 // SimpleMessageListenerContainer 设置
 MessageListenerAdapter adapter = new MessageListenerAdapter(new MessageDelegate());
 container.setMessageListener(adapter);
```

```shell
# 方法名为什么要写死为 handleMessage 的源码依据:

# 可以自定义修改方法名 (根据源码提供 set 方法)
public void setDefaultListenerMethod(String defaultListenerMethod) {
		this.defaultListenerMethod = defaultListenerMethod;
}

# 也可以添加一个转换器: 从字节数组转换为 String
	# （当前SpringBoot 版本2.1.9），默认的消息转换器即 SimpleMessageConverter
	# 已经将其转为 String
	
# 队列名称和方法名称也可以进行 一一匹配
adapter.setQueueOrTagToMethodName(Map<String, String>);
# 例如:
map.put("queue1", "method1");
```

![UTOOLS1578627312430.png](http://yanxuan.nosdn.127.net/bdffce009f6c4fca078c9deb988e0732.png)

### MessageConverter

```shell
# 消息转换器

# 在发送消息时，正常情况下消息体为二进制的数据方式进行传输
	# 如果希望内部帮我们进行转换，或者指定自定义的转换器，就需要用到 MessageConverter
	
# 自定义常用转换器:
	# 一般来讲都要实现这个接口 MessageConverter
	# 重下下面两个方法:
		# toMessage: Java 对象转换为 Message
		# fromMessage: Message 对象转换为 Java 对象
		
# 常用转换器:
	# Jackson2jsonMessageConverter: 进行 Java 对象与 JSON 对象的转换
	# DefaultJackson2JavaTypeMapper: 映射器，可以进行 Java 对象的映射关系
	
# 自定义二进制转换器:
	# 比如 图片、PDF、PPT、流媒体
	
# 全局转换器:
	# 往里面加各种各样的转换器就好了
ContentTypeDelegatingMessageConverter
```

```shell
# 此处代码略...（感觉没有太大记录的必要）
```

![UTOOLS1578648715437.png](http://yanxuan.nosdn.127.net/ffc101bba52e3664274774fc6afc4f0c.png)

## RabbitMQ 与 SpringBoot2.0 整合实战

### 依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

### 生产端

#### 配置

```yaml
spring:
  rabbitmq:
    addresses: 127.0.0.1:5672 # 地址
    username: guest	# 用户名
    password: guest # 密码
    virtual-host: / # 虚拟主机
    connection-timeout: 15000 # 连接超时时间
    # 实现一个监听器用于监听 Broker 端给我们返回的确认请求
    # RabbitTemplate.ConfirmCallback
    publisher-confirms: true	
    # 保障消息对 Broker 端是可达的，如果出现路由键不可达的情况
    # 则使用监听器对不可达的消息进行后续的处理，保证消息的路由成功
    publisher-returns: true
    # 注意: 在发送消息的时候对 template 进行配置 mandatory 为 true 保证监听有效
    template:
      mandatory: true
 
# 生产端还可以配置其他属性，比如发送重试，超时时间，次数，间隔等
```

#### 消息发送的代码

```java
@Component
public class RabbitSender {

    @Autowired
    private RabbitTemplate rabbitTemplate;

    /**
     * 回调函数: confirm 确认
     * correlationDate : 全局唯一消息凭证
     * ack : 消息是否签收 : 根据业务自己决定消息成功后修改 Msg 表记录的状态，或者失败的补偿机制
     * cause : 失败原因
     */
    final RabbitTemplate.ConfirmCallback confirmCallback = ((correlationData, ack, cause) -> {
        System.err.println("correlationDate: " + correlationData);
        System.err.println("ack: " + ack);
        System.err.println("cause: " + cause);
        if (!ack) System.err.println("消息异常处理...");
    });

    /**
     * 回调函数: return 返回
     * message : 消息内容
     * replyCode : 重试的错误码
     * replyText : 重试的提示
     * exchange : 交换机
     * routingKey : 发送消息的路由
     */
    final RabbitTemplate.ReturnCallback returnCallback = (((message, replyCode, replyText, exchange, routingKey) -> {
        System.err.println("return exchange: " + exchange + ", routingKey: "
                + routingKey + ", replyCode: " + replyCode + ", replyText: " + replyText
        + ", message: " + message);
    }));

    /**
     * 发送消息: 构建 Message 对象
     *  可重载方法为发送对象消息，内部由 MessageConverter 进行 JSON 转换
     * @param message   : 消息体
     * @param properties    : 消息属性
     */
    public void send(Object message, Map<String, Object> properties) {
        MessageHeaders headers = new MessageHeaders(properties);
        Message<Object> msg = MessageBuilder.createMessage(message, headers);
        rabbitTemplate.setConfirmCallback(confirmCallback);
        rabbitTemplate.setReturnCallback(returnCallback);
        // 自己生成全局唯一的消息Id
        CorrelationData data = new CorrelationData("12345");

        rabbitTemplate.convertAndSend("exchangeName","routingKey", msg, data);
    }

}
```

### 消费端

#### 消费端的核心配置

```yaml
spring:
  rabbitmq:
    addresses: 127.0.0.1:5672
    username: guest
    password: guest
    virtual-host: /
    connection-timeout: 15000
    listener:
      simple:
        acknowledge-mode: manual  # 手工签收ack
        concurrency: 1  # 默认消费限制为1
        max-concurrency: 5 # 最大消费限制为5
        
# 首先配置手工确认模式，用于 ACK 的手工处理
	# 这样才可以保证消息的可靠性送达
	# 或者在消费端消费失败的时候可以做到重回队列、根据业务记录日志等处理
	
# 可以设置消费端的监听个数和最大个数，用于控制消费端的并发情况
```

```shell
# @RabbitListener 注解使用:
	# 消费端 @RabbitMQListener 注解，在实际工作中非常的好用
	
# @RabbitListener 是一个组合注解，里面可以注解配置:
	# @QueueBinding、@Queue、@Exchange 
	# 直接通过这个组合注解一次性搞定消费端交换机、队列、绑定、路由、并且配置监听功能等
```

#### 消费端的代码

```java
@Component
public class RabbitReceiver {
  
	@RabbitListener(bindings = @QueueBinding(
			value = @Queue(value = "queue-1", 
			durable="true"),
			exchange = @Exchange(value = "exchange-1", 
			durable="true", 
			type= "topic", 
			ignoreDeclarationExceptions = "true"),
			key = "springboot.*"
			)
	)
	@RabbitHandler
	public void onMessage(Message message, Channel channel) throws Exception {
		System.err.println("--------------------------------------");
		System.err.println("消费端Payload: " + message.getPayload());
		Long deliveryTag = (Long)message.getHeaders().get(AmqpHeaders.DELIVERY_TAG);
		//手工ACK
		channel.basicAck(deliveryTag, false);
	}
	
	/**
	 * 
	 * 	spring.rabbitmq.listener.order.queue.name=queue-2
		spring.rabbitmq.listener.order.queue.durable=true
		spring.rabbitmq.listener.order.exchange.name=exchange-1
		spring.rabbitmq.listener.order.exchange.durable=true
		spring.rabbitmq.listener.order.exchange.type=topic
		spring.rabbitmq.listener.order.exchange.ignoreDeclarationExceptions=true
		spring.rabbitmq.listener.order.key=springboot.*
	 * @param order
	 * @param channel
	 * @param headers
	 * @throws Exception
	 */
	@RabbitListener(bindings = @QueueBinding(
			value = @Queue(value = "${spring.rabbitmq.listener.order.queue.name}", 
			durable="${spring.rabbitmq.listener.order.queue.durable}"),
			exchange = @Exchange(value = "${spring.rabbitmq.listener.order.exchange.name}", 
			durable="${spring.rabbitmq.listener.order.exchange.durable}", 
			type= "${spring.rabbitmq.listener.order.exchange.type}", 
			ignoreDeclarationExceptions = "${spring.rabbitmq.listener.order.exchange.ignoreDeclarationExceptions}"),
			key = "${spring.rabbitmq.listener.order.key}"
			)
	)
	@RabbitHandler
	public void onOrderMessage(@Payload Order order, 
			Channel channel, 
			@Headers Map<String, Object> headers) throws Exception {
		System.err.println("--------------------------------------");
		System.err.println("消费端order: " + order.getId());
		Long deliveryTag = (Long)headers.get(AmqpHeaders.DELIVERY_TAG);
		//手工ACK
		channel.basicAck(deliveryTag, false);
	}
	
}
```

## 略过的章节

```shell
# RabbitMQ 与 SpringCloud Stream 整合实战

# RabbitMQ 集群架构模式 - 准备模式（Warren)

# RabbitMQ 集群架构模式 - 镜像模式（Mirror）

# RabbitMQ 集群架构模式 - 多活模式（Federation）

# RabbitMQ 集群镜像队列构建实现可靠性存储

# RabbitMQ 集群整合负载均衡构建实现可靠性存储

# RabbitMQ 集群整合负载均衡基础组件 HaProxy

# RabbitMQ 集群整合高可用组件 KeepAlived

# RabbitMQ 集群配置文件详解

# RabbitMQ 集群恢复与故障转移的 5 种解决方案

# RabbitMQ 集群延迟队列插件应用

# BAT、TMD 大厂单元化架构设计衍变之路分享

# SET 化架构设计策略（异地多活架构）

# SET 化架构设计原则

# SET 化消息中间件架构实现

# 一线大厂的 MQ 组件实现思路和架构设计思路

# 基础 MQ 消息组件设计思路（迅速，确认，批量，延迟）

# 基础 MQ 消息组件设计思路（顺序）

# 基础 MQ 消息组件设计思路（事务）

# 消息幂等性保障 - 消息路由规则架构设计思路
```

