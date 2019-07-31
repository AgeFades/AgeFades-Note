# 慕课 - RabbitMq

## 主流消息中间件介绍

### 	ActiveMq

​		Apache 出品，完全支持 JMS 规范

​		丰富 Api，多种集群构建模式

​		MQ衡量指标 : 服务性能、数据存储、集群架构

#### 		集群模式

![UTOOLS1562037240304.png](https://i.loli.net/2019/07/02/5d1acbfd6cb4950746.png)

### 	KafKa

​		Apache顶级项目

​		基于 Pull 的模式来处理消息消费，追求高吞吐量

​		开始目的是用于日志收集和传输

​		0.8版本开始支持复制，不支持事务

​		对消息的重复、丢失、错误没有严格要求。

​		适合产生大量数据的互联网服务的数据收集业务

#### 		集群模式

![UTOOLS1562037556900.png](https://i.loli.net/2019/07/02/5d1acd397adbb12195.png)

### 	RocketMq

​		Apache 顶级项目，纯 Java 开发

​		高吞吐、高可用、适合大规模分布式系统应用

​		起源 Kafka，对消息的可靠传输及事务性做了优化

#### 		集群模式

![UTOOLS1562037749071.png](https://i.loli.net/2019/07/02/5d1acdf8b599a38249.png)

### 	RabbitMq

​		Erlang 语言开发，基于 AMQP 协议

​		面向消息、队列、路由、可靠性、安全。

​		对数据一致性、稳定性、可靠性要求高，对性能和吞吐量的要求低

#### 		集群模式

![UTOOLS1562038507116.png](https://i.loli.net/2019/07/02/5d1ad0ec09bd048297.png)

## RabbitMq 核心概念以及 AMQP 协议

### 	RabbitMq 高性能的原因

​		Erlang 语言最初用于交换机领域的架构模式，在 Broker 之间进行数据交互的性能非常优秀。

​		Erlang 有和原生 Socket 一样的延迟

### 	AMQP 高级消息队列协议

​		具有现代特征的二进制协议。

​		提供统一消息服务的应用层标准协议

​		应用层协议的一个开放标准，面向消息的中间件设计

#### 		协议模型

![UTOOLS1562040096920.png](https://i.loli.net/2019/07/02/5d1ad722cb89f39254.png)

### ​	AMQP 核心概念

#### 		Server :

​			又称 Broker ，接收客户端的连接，实现 AMQP 实体服务

#### 		Connection : 

​			连接，应用程序与 Broker 的网络连接

#### 		Channel :

​			网络信道，所有的操作都在 Channel 中进行。

​			客户端可建立多个 Channel，每个 Channle 代表一个会话任务

#### 		Message :

​			服务器和应用程序之间传送的数据。

​			由 Properties 和 Body 组成。

​			Properties 可以对消息进行修饰，比如优先级、延迟等高级特性。

​			Body 是消息体内容

#### 		Virtual host :

​			虚拟地址，用于进行逻辑隔离，最上层的消息路由。

​			一个 Virtual host 里面可以有若干个 Exchange 和 Queue

​			同一个 Virtual Host 里面不能有相同名称的 Exchange 或 Queue

#### 		Exchange :

​			交换机，接收消息，根据路由键转发消息掉绑定的队列。

#### 		Binding :

​			Exchange 和 Queue 之间的虚拟连接，binding 中可以包含 routing key

#### 		Routing key :

​			一个路由规则，虚拟机可用它来确定如何路由一个特定消息

#### 		Queue :

​			消息队列，保存消息并将它们转发给消费者

### 	RabbitMQ 消息流转图解

![ZG5Au8.png](https://s2.ax1x.com/2019/07/02/ZG5Au8.png)

## RabbitMq 安装与使用

​	此处本人使用 docker 一键安装，略。

```shell
docker run -d --hostname rabbitmq --name rabbitmq -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 rabbitmq:3-management
```

​	官网地址 : http://www.rabbitmq.com

​	默认账号密码 : guest		

​	默认访问地址 : localhost:15672

### 	命令行与管控台

#### 		命令行

```shell
rabbitmqctl  # 服务命令前缀，具体命令例 : rabbitmqctl stop_app

​				stop_app # 关闭应用 

​				start_app # 启动应用

​				status # 节点状态

​				add_user username password # 添加用户

​				list_users # 列出所有用户

​				delete_user username # 删除用户

​				clear_permissions -p vhostpath username # 清除用户权限

​				list_user_permissions username # 列出用户权限

​				change_password username newpassword # 修改密码

​				set_permissions -p vhostpath username ".*" ".*"".*" # 设置用户权限

​				add_vhost vhostpath # 创建虚拟主机

​				list_vhosts # 列出所有虚拟主机

​				list_permissions -p vhostpath # 列出虚拟主机上所有权限
		
				list_queues # 查看所有队列消息
				
				-p vhostpath purge_queue blue # 清除队列里的消息
				
				reset # 移除所有数据，在stop_app 之后使用
				
				join_cluster <clusternode> [--ram] # 组成集群命令
					
				change_cluster_node_type disc | ram # 修改集群节点的存储形式
					
				forget_cluster_node [--offline] # 忘记节点(摘除节点)
						
				rename_cluster_node oldnode1 newnode1 [oldnode2 ...] # 修改节点名称
			
```

#### 		控制台

​			略	

#### 		使用

​			本章节讲述最低层次的 Client 连接，略

### 		Exchange

​			![UTOOLS1562045418769.png](https://i.loli.net/2019/07/02/5d1aebebb06a389723.png)

#### 			交换机属性

​				Name : 交换机名称

​				Type : 交换机类型

​					direct : 

​						所有消息将被转发到 RoutingKey 中指定的 Queue

​						可以使用 RabbitMQ 自带的 deafult Exchange

​						所以不需要将 Exchange 进行任何绑定操作

​						消息传递时，RoutingKey 必须完全匹配才会被队列接收，否则被抛弃。

![UTOOLS1562046238390.png](https://i.loli.net/2019/07/02/5d1aef1f13db676835.png)

​					topic : 

​						所有消息被转发到所有关心 RoutingKey 中指定 Topic 的Queue 上

​						Exchange 将 RoutingKey 和某 Topic 进行模糊匹配，此时队列需要绑定一个Topic

![UTOOLS1562046457601.png](https://i.loli.net/2019/07/02/5d1aeffa48f2d42945.png)

​					![UTOOLS1562046497891.png](https://i.loli.net/2019/07/02/5d1af0223073455845.png)

​					fanout :

​						不处理路由键，只需要简单的将队列绑定到交换机上

​						发送到交换机的消息都会被转发到与该交换机绑定的所有队列上

​						转发消息最快

![UTOOLS1562046668980.png](https://i.loli.net/2019/07/02/5d1af0cd2a46751569.png)

​					headers :

​						很少使用，略

​				Durability : 是否需要持久化（true | false）

​				Auto Delete : 当最后一个绑定到 Exchange 上的队列删除后，自动删除该 Exchange

​				Internal : 当前 Exchange 是否用于 RabbitMQ 内部使用，默认为 false

​				Arguments : 扩展参数，用于扩展 AMQP 协议自制定化使用				

### 	Queue

​		消息队列，实际存储消息数据

#### 		属性

​			Durability : 是否持久化

​				Durable : 是

​				Transient : 否

​			Auto delete : 当最后一个监听被移除后，该 Queue 会自动被删除

### 	Message

​		服务器和应用程序之间传送的数据

​		由 Properties 和 Payload（Body）组成

#### 		属性

​			delivery mode

​			headers : 自定义属性

​			content_type : 消息类型

​			content_encoding : 消息编码

​			priority : 优先级	

​			correlation_id :

​			reply_to

​			expiration : 过期时间

​			message_id

​			timestamp

​			type

​			user_id

​			app_id

​			cluster_Id		

## RabbitMQ 高级特性

### 	保障消息 100% 投递成功

#### 		生产端的可靠性投递

​			保障消息的成功发出

​			保障 MQ 节点的成功接收

​			发送端收到 MQ 节点（Broker）确认应答

​			完整的消息进行补偿机制

##### 			解决方案

​				消息入库，对消息状态进行打标

![UTOOLS1562048795580.png](https://i.loli.net/2019/07/02/5d1af920c408a57868.png)

​				消息的延迟投递，做二次确认，回调检查

![UTOOLS1562049119850.png](https://i.loli.net/2019/07/02/5d1afa631d62861758.png)

### 	幂等性概念

#### 		消费端 - 幂等性保障

​			避免消息的重复消费问题

##### 			解决方案

​				唯一ID + 指纹码机制，利用数据库主键去重

​				利用 Redis 的原子性去实现

### 	Confirm 确认消息

​		消息的确认，是指生产者投递消息后，如果 Broker 收到消息，则会给生产者一个应答。

​		生产者进行接收应答，用来确定这条消息是否正常的发送到 Broker。

​		是消息的可靠性传递的核心保障。	

#### 		流程图	

![UTOOLS1562050586388.png](https://i.loli.net/2019/07/02/5d1b001b567bb18210.png)

#### 		开启步骤

​			在 channel 上开启确认模式，channel.confirmSelect()

​			在 channel 上添加监听，addConfirmListener

​				监听成功或失败的返回结果，进行重试或日志记录等操作

### 	Return 消息机制

​		Return Listener 用于处理一些不可路由的消息

#### 		配置

​			Mandatory : 

​				true : 监听器接收路由不可达的消息，然后进行后续处理

​				false : broker 端自动删除该消息

#### 		流程

![UTOOLS1562050927376.png](https://i.loli.net/2019/07/02/5d1b0171527ca78200.png)

### 	消费端限流

​		RabbitMQ 提供了一种 qos（服务质量保障）功能

​		在非自动确认消息的前提下，如果一定数目的消息（通过基于 consumer 或者 channal 设置 Qos 的值）

​			未被确认前，不进行消费新的消息。

```java
/**
 * 消费端
 * prefetchSize : 消息大小限制(一般为0 不限制)
 * prefetchCount : 一次能够处理的消息数量(一般为1)
 * global : true 作用于 channel 通道级，false 作用于 Consumer 级别
 */
void BasicQos(uint prefetchSize,ushor prefetchCount,bool global);
```

### ​	消费端 ACK 与重回队列

#### 		消费端的手工 ACK 和 NACK

​			消费端进行消费的时候，如果由于业务异常我们可以进行日志的记录，然后进行补偿

​			如果由于服务器宕机等严重问题，那就需要手工 ACK 保障消费端消费成功

#### 		消费端的重回队列

​			消费端重回队列是为了对没有处理成功的消息，把消息重新回递给Broker

​			一般实际应用中，都会关闭重回队列

### 	TTL 队列/消息

​		TTL : 生存时间

​		RabbtitMQ 支持消息的过期时间，在消息发送时可以进行指定

​		RabbitMQ 支持队列的过期时间，从消息入队开始计算，只要超过了队列的超时时间，消息自动清除	

### 	死信队列

​		当消息在一个队列中变成死信之后，能被重新 publish 到另一个 Exchange（DLX）

#### 		死信条件

​			消息被拒绝（basic.reject / basic.nack）并且 requeue = false

​			消息 TTL 过期

​			队列达到最大长度

#### 		DLX

​			一个正常的 Exchange，能在任何的队列上被指定，实际上就是设置某个队列的属性。

​			当该队列有死信时，RabbitMQ 自动将该死信发布到设置的 Exchange 上，进而被路由到另一个队列

​			可以监听该队列中消息做相应的处理

#### 		设置

​			首先需要设置死信队列的 exchange 和 queue，然后进行绑定

​				Exchange : dlx.exchange

​				Queue : dlx.queue

​				RoutingKey : #

​			然后正常声明交换机、队列、绑定

​				在队列加上一个参数 : arguments.put("x-dead-letter-exchange","dlx.exchange");

​			这样消息在过期、requeue、队列达到最大长度时，消息就可以直接路由到死信队列

## RabbitMQ 高级整合应用

### 	RabbitMQ 整合 Spring AMQP

#### 		核心组件

​			RabbitAdmin	

​			SpringAMQP 声明

​			RabbitTemplate

​			SimpleMessageListenerContainer

​			MessageListenerAdapter

​			MessageConverter

#### 		开发流程

```xml
   <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>

```

```yaml
spring:
	rabbitmq:
    	host: x
    	port: x
   		username: x
    	password: x
    	virtualHost: /
```

```java
/**
 * RabbitMQ 配置类
 */
@Configuration
@ComponentScan({xxx});	// 扫包
public class RabbitMqConfig{
    
    @Bean
    // 声明要注入的属性前缀，SpringBoot会自动把相关属性通过set方法注入到DataSource中
    @ConfigurationProperties(prefix = "spring.rabbitmq")
    public ConnetcionFactory connectionFactory(){
      	CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
        return connectionFactory;
    }
    
    /**
	 * 操作 RabbitMQ
	 * 底层实现就是从 Spring IOC 中获取 Exchange、Bingding、RoutingKey 以及 Queue 的 @Bean
 	 */
    @Bean
    public RabbitAdmin rabbitAdmin(ConnectionFactory connectionFactory){
        RabbitAdmin rabbitAdmin = new RabbitAdmin(connectionFactory);
        rabbitAdmin.setAutoStartup(true);	// 必须设置，否则 Spring IOC 不加载该 Bean
        return rabbitAdmin;
    }
    
    /**
     * 消息模板类
     */
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
    	RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
    	return rabbitTemplate;
    }
    
    /**
     * 消息监听容器
     */
     @Bean
    public SimpleMessageListenerContainer messageContainer
        									(ConnectionFactory connectionFactory) {
    	
    	SimpleMessageListenerContainer container =
            			new SimpleMessageListenerContainer(connectionFactory);
    	container.setQueues("QueueName","xxx"); // 监听队列，支持多个
    	container.setConcurrentConsumers(1); // 当前消费者数量
    	container.setMaxConcurrentConsumers(5);	// 最大消费者数量
    	container.setDefaultRequeueRejected(false);	// 不重回队列
    	container.setAcknowledgeMode(AcknowledgeMode.AUTO);	// 自动签收 ACK
    	container.setExposeListenerChannel(true);
    	container.setConsumerTagStrategy(queue -> System.out.println());// 消费端的标签策略
        // ... 不再记录，需要请看源码说明
    
}
```

```java
public class Test(){
    @Autowired
    private RabbitAdmin rabbitAdmin;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
   
    public void test() throws Exception{
        // 创建 Exchange, topic | fanout .. 不做展示
        rabbitAdmin.declareExchange(new DirectExchange
                                    ("ExchangeName","持久化与否","自动删除与否"));   
        
        // 创建 Queue
        rabbitAdmin.declareQueue(new Queue("QueueName","持久化与否"));
        
        // 创建 Binding
        rabbitAdmin.declareBinding(new Binding
                                   ("QueueName",Binding.DestinationType.QUEUE,
                                   "ExchangeName","ExchangeType",new Hashmap<>()));
        
        // 创建 Binding
        rabbitAdmin.declareBinding(BindingBuilder
                                  .bind(new Queue("QueueName"),xx))
            					  .to(new TopicExchange("ExchangeName",xx,xx))
            					  .with("user.#"); // 自定义路由规则
        
        // 清空队列数据
        rabbitAdmin.purgeQueue("QueueName",xx);
        
        // 发送消息
        MessageProperties messageProperties = new MessageProperties();	// 消息属性对象
        messageProperties.getHeaders().put("desc","xxx");	// Headers 本质 Map，存放 KV
        Message message = new Message("内容".getBytes(),messageProperties);  // 消息对象
        
        rabbitTemplate.convertAndSend("ExchangeName","RoutingKey",message,message -> {
            // 对消息进行额外设置，例
            message.getMessageProperties().getHeaders().put("desc","修改desc");
            // ...
        })
            
        // 简单版
		rabbitTemplate.convertAndSend("ExchangeName","RoutingKey","消息内容");	
    }    
       		
}
```

##### 			MessageListenerAdapter ：消息监听适配器

​	通过 MessageListenerAdapter 的代码看出如下核心属性 :

​		defaultListenerMethod 默认监听方法名称，用于设置监听方法名称

​		Delegate 委托对象，实际的委托对象，用于处理消息

​		queueOrTagToMethodName 队列标识与方法名称组成的集合

​			可以一一进行队列与方法名称的匹配

​			队列和方法名称绑定，即指定队列里的消息会被绑定的方法所接受处理

##### 			MessageConverter : 消息转换器

​	正常情况下，消息体为二级制进行传输

​	自定义常用转换器 : MessageConverter ，一般来说都需要实现该接口

​	重写两个方法 :

​		toMessage : Java 对象转换为 Message 对象

​		fromMessage : Message 对象转换为 Java 对象

​	Jackson2JsonMessageConverter : 进行 Java 对象的转换功能

​	DefaultJackson2JavaTypeMapper : 进行 Java 对象的映射关系

​	自定义二进制转换器 : 比如图片类型、PDF、PPT、流媒体

### SpringBoot 整合配置详解

#### 生产端

```java
/**
 * 注意 : 在发送消息的时候对 template 进行配置 mandatory=true 保证监听有效
 * spring:
 *   rabbitmq:
 *		publisher-confirms=true
 * 		publisher-returns=true
 *		template:
 *			mandatory=true
 * 生产端还可以配置其他属性，比如发送重试、超时时间、次数、间隔等
 */
publisher-confirms // 实现一个监听器用于监听 Broker 端给我们返回的确认请求
    RabbitTemplate.ConfirmCallback
    rabbitTemplate.setConfirmCallback( (correlationData,ack,cause) -> 
     if(!ack) log.info("消息发送异常");
    )
    
publisher-returns // 保证消息对 Broker 端是可达的，如果出现路由键不可达的情况
				  // 则使用监听器对不可达的消息进行后续的处理，保证消息的路由成功
	RabbitTemplate.ReturnCallback
    rabbitTemplate.setReturnCallback( 
    				(message,replyCode,replyText,exchange,routingKey) -> {
    		System.out.println("消息返回结果")
    })
			
	
```

#### 消费端

```yaml
spring:
	rabbitmq:
		listener:
			simple:
				acknowledge-mode: MANUAL # 手动签收 ACK
				concurrency: 1
				max-concurrency: 5
```

​	首先配置ACK手工确认模式，保证消息的可靠性送达

​	或者在消费端消费失败的时候可以做到重回队列、根据业务记录日志等处理

​	可以设置消费端的监听个数和最大个数，用于控制消费端的并发情况

​	消费端监听 @RabbitMQListener 注解

​	@RabbitListener 是一个组合注解，里面可以注解配置

​		@QueueBinding

​		@Queue

​		@Exchange

​		直接通过这个组合注解一次性搞定消费端交换机、队列、绑定、路由、并且配置监听功能等

![UTOOLS1564468404074.png](https://i.loli.net/2019/07/30/5d3fe4b65cf5d22354.png)

### Spring Cloud Stream 整合

![UTOOLS1564468961690.png](https://i.loli.net/2019/07/30/5d3fe7067057a51243.png)

## RabbitMQ 集群架构

### 主备模式

​	也被称为 Warren 模式

​	一般在并发和数据量不高的情况下

​	主节点挂了，从节点提供服务（类似 ActiveMq 利用 Zookeeper 做主备）

```shell
# HA Proxy 配置
listen rabbitmq_cluster
bind 0.0.0.0:5673 # 配置 tcp 监听
mode tcp # 轮询
balance roundrobin # 负载均衡
server xxx ip:port check inter 5000 rise 2 fall 2	# 主节点
server xxx ip:port backup check inter 5000 rise 2 fall 2 # 从节点
# inter 每隔 5s 对 MQ 集群做健康检查，2次正确证明服务器可用，2次失败证明服务不可用，并且配置主从切换
```

### 远程模式

​	实现双活的一种模式，简称 Shovel 模式

​	把消息进行不同数据中心的复制工作，跨地域的让两个 mq 集群互联

### 镜像模式

​	Mirror 模式

​	保证 100% 数据不丢失

​	主要就是实现数据的同步（100% 一般是 3 个节点以上）

![UTOOLS1564470564040.png](https://img01.sogoucdn.com/app/a/100520146/c66c4cc9dd36b10d11fab49b49e737d9)

### 多活模式

​	实现异地数据复制的主流模式

​	依赖 rabbitmq 的 federation 插件

​		federation 插件是个不需要构建 Cluster，而在 Brokers 之间传递消息的高性能插件

​		连接的双方可以使用不同的 users 和 virtual hosts

​		也可以使用版本不同的 RabbitMQ 和 Erlang

​	RabbitMQ 部署架构采用多中心模式，在每套数据中心各部署一套 RabbitMQ 集群，各中心的 RabbitMQ 服务除了需要为业务提供正常的消息服务外，中心之间还需要实现部分队列消息共享。

![UTOOLS1564471516040.png](https://img01.sogoucdn.com/app/a/100520146/f231c0700be316d4f26d7fa614161da1)

### RabbitMQ 集群镜像模式构建

![UTOOLS1564473776486.png](https://img04.sogoucdn.com/app/a/100520146/36bd9af05308ac2d0f256a6cb58c437d)

```shell
# 1. 停止各个节点的 MQ 服务
rabbitmqctl stop

# 2. 文件同步
cd /var/lib/rabbitmq
chmod 777 .erlang.cookie # master 节点
scp .erlang.cookie hort:/var/lib/rabbitmq/
vim /etc/hostname # 修改当前主机名

# 3. 启动服务
cd /usr/local
rabbitmq-server -detached
lsof -i:port # 检测 mq 服务端口是否启动

# 4. slave 加入集群
rabbitmqctl stop_app
rabbitmqctl join_cluster rabbit@hostname # master 的 hostname
rabbitmqctl start_app

# rabbitmqctl forget_cluster_node rabbit@hostname 移除 xx 节点

# 5. 修改集群名称
rabbitmqctl set_cluster_name xxx # 默认为 master 节点名称

# 6. 查看集群状态
rabbitmqctl cluster_status # 查看集群状态

# 7. 同步消息 master
rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'

```

### HAProxy

​	HAProxy 是一款提供高可用性、负载均衡以及基于 TCP（第四层）和 HTTP（第七层）应用的代理软件，支持虚拟主机。

#### 	Haproxy 性能最大化

​		单进程、事件驱动模型显著降低了上下文切换的开销及内存占用

​		任何可用情况下，单缓冲机制能以不复制任何数据的方式完成读写操作，节约大量 CPU and Memory

​		借助于 Linux 2.6 以上的 splice() 系统调用，HAProxy 可以实现 **零复制转发**，在 3.5 以上的 OS 中还可以实现 **零复制启动**

​		内存分配器在固定大小的内存池中可实现即时内存分配，显著减少创建一个会话的时长

​		侧重于使用弹性二叉树、实现 O(logN) 的低开销来保持计时器命令、保持运行队列命令以及管理轮询及最少连接队列	

#### 	Haproxy 安装

```shell
yum install gcc vim wget # 下载依赖包

wget http://www.haproxy.org/download/1.6/src/haproxy-1.6.5.tar.gz # 下载源码包

tar -zxvf haproxy-1.6.5.tar.gz -C /usr/local # 解压

cd /usr/local/haproxy-1.6.5 # 进入目录

make TARGET=linux31 PREFIX=/usr/local/haproxy # 编译
make install PREFIX=/usr/local/haproxy # 安装
mkdir /etc/haproxy

groupadd -r -g 149 haproxy # 赋权
useradd -g haproxy -r -s /sbin/nologin -u 149 haproxy

vim /etc/haproxy/haproxy.cfg # 创建配置文件	
# 具体配置略

/usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg # 启动 haproxy
ps -ef | grep haproxy # 查看进程状态

```

### KeepAlived

​	主要通过 VRRP 协议实现高可用功能的（虚拟路由器冗余协议）

​	VRRP 出现的目的就是为了解决静态路由单点故障问题，保证个别节点宕机时，整个网络不间断地运行

​	KeepAlived 一方面具有配置管理 LVS 的功能，同时具有对 LVS 下面节点进行健康检查的功能，另一方面也可以实现系统网络服务的高可用功能

#### 	KeepAlived 高可用原理

​		KeepAlived 正常工作时，主节点不断向从节点发送心跳消息

​		主节点不发送心跳时，从节点调用自身接管协议，接管主节点的 ip 资源及服务

​		主节点恢复时，从节点释放主节点 Ip 资源及服务，回到从节点角色

```shell
yum install -y openssl openssl-devel # 安装所需软件包

wget http://www.keepalived.org/software/keepalived-1.2.18.tar.gz # 下载

tar -zxvf keepalived-1.2.18.tar.gz -C /usr/local/ # 解压

cd keepalived-1.2.18/ && ./config --prefix=/usr/local/keepalived

make && make install # 编译、安装

# 将 keepalived 安装成 Linux 系统服务
mkdir /etc/keepalived
cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/

cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/init.d/
cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
ln -s /usr/local/sbin/keepalived /usr/sbin/

# 如果存在则删除 rm /sbin/keepalived
ln -s /usr/local/keepalived/sbin/keepalived /sbin/

# 设置开机自启
chkconfig keepalived on 

vim /etc/keepalived/keepalived.conf # 修改配置文件
# 具体配置略
```

### 集群配置文件

#### ​	关键配置参数

```shell
tcp_listerners # 设置 rabbitmq 的监听端口，默认为 [5672]

dist_free_limit # 磁盘低水位线，若磁盘容量低于指定值则停止接收数据，默认值为{mem_relative,1.0}，即与内存相关联 1:1，也可定制为多少 byte

vm_memory_high_watermark # 设置内存低水位线，若低于该水位线，则开启流控机制，默认值 0.4，即内存总量的 40%

hipe_compile # 将部分 rabbitmq 代码用 High Performance Erlang compiler 编译，提升性能，该参数是实验性，若出现 erlang vm segfaults ，应关掉

force_fine_statistics # 若为 true，更加精细化的统计，影响性能

# 集群节点模式 : Disk 为磁盘模式存储，Ram 为内存模式存储
# 具体配置略
```

### 集群回复与故障转移​

#### 	前提

​		AB 两个节点组成一个镜像队列，B为 Master

#### 	场景1

​		A先停，B后停

##### 		方案

​			先启动 Master，再启动 Slave  or 先启动 Slave，30S 之内启动 Master 即可恢复镜像对列

#### 	场景2

​		AB 同时停机

##### 		方案

​			30s 之内连续启动 AB 即可恢复镜像队列

#### 	场景3

​		A先停、B后停、且A无法恢复

##### 		方案

​			启动 B ，在 B 上调用控制台命令 rabbitmqctl forget_cluster_node A

​			解除与 A 的关系，再将新的 Slave 节点加入B 即可

#### 	场景4 	

​		A先停、B后停、且B无法恢复	

##### 		方案

​			3.4.2 版本后，rabbitmqctl forget_cluster_node --offline B

​			解除与 B 的关系，再将新的 Slave 节点加入A 即可

#### 	场景5

​		A先停、B后停、且AB无法恢复，但是能得到 A 或 B 的磁盘文件

##### 		方案

​			将 A 或 B 的磁盘文件拷贝到新节点的对应目录下，将新节点的 Hostname 改为 A 或 B，启动后按场景3 或 场景4 处理

### 延迟插件的作用

#### 	延迟队列可以做什么事情

​		消息的延迟推送

​		定时任务的执行

​		消息重试策略的配合使用

​		业务削峰限流、降级的异步延迟消息机制

#### 	延迟插件的安装

​		过于繁琐，略

## 互联网大厂 SET 架构

​	SET : 单元化	

​	在 RabbitMQ 中实现 SET 架构，就是借助前面所述 "多活" 模式

## 大厂的 MQ 组件实现思路和架构设计方案

![UTOOLS1564544543927.png](https://i.loli.net/2019/07/31/5d410e220a1fa45071.png)		

### MQ 组件实现功能点

​		支持消息高性能的序列化转换、异步化发送消息

​		支持消息生产实例与消费实例的链接化缓存化、提升性能

​		支持可靠性传递消息，保障消息的100%不丢失

​		支持消费端的幂等操作，避免消费端重复消费的问题

​		支持迅速消息发送模式，在一些 日志收集/统计分析 等需求下可以保证高性能，超高吞吐量

​		支持延迟消息模式，消息可以延迟发送，指定延迟消息，用于某些延迟检查，服务限流场景

​		支持事物消息，且 100% 保障可靠性投递

​		支持顺序消息，保证消息送达消费端的前后顺序

​		支持消息补偿，重试，以及快速定位异常/失败消息

​		支持集群消息负载均衡

​		支持消息路由策略

### 迅速消息发送

​	迅速消息是指消息不进行落库存储，不做可靠性保证

​	适用非核心消息、日志数据、统计分析等

​	性能最高，吞吐量最大

### 确认消息发送

![UTOOLS1564545157886.png](https://i.loli.net/2019/07/31/5d41108732fea10585.png)

​	业务数据、消息数据入库

​	发送消息

​	消费端执行业务逻辑，成功后手动 ack

​	生产端收到 ack，confirmCallBack 修改数据库中消息记录状态

​	如果消费端消费失败，未发送 ack

​	定时任务定时检测消息表中状态为 0 的消息，抓取数据并重发，记录次数

​	连续3次重试失败的消息，将消息状态修改为 2 ，等待人工补偿

### 批量消息发送

​	将消息放到一个集合里统一进行提交

​	比如投掷到 threadloacl 里的集合，拥有相同会话 Id,并带有此次提交消息的相关属性，最重要的是将这一批消息合并，对于 Channel 而言，就是发送一次消息

​	在消费端进行批量处理，当成原子业务操作

​	不保证可靠性

![UTOOLS1564550049351.png](https://i.loli.net/2019/07/31/5d4123a33815460714.png)	

### 延迟消息发送

​	在 Message 封装的时候添加 delayTime 属性

### 顺序消息

​	发送的顺序消息，必须保障消息投递到同一个队列中，且该消费者只有一个

​	统一提交，并且所有消息的会话ID 一致

​	添加消息属性 : 顺序标记的序号、和本次顺序消息的 SIZE 属性，进行落库操作

​	并行进行发送给自身的延迟消息，进行后续处理消费

​	消费端收到延迟消息后，根据会话 ID、SIZE 抽取数据处理即可

​	定时轮询补偿机制

![UTOOLS1564550707582.png](https://i.loli.net/2019/07/31/5d412635606b775639.png)

### 事务消息发送

​	采用类似可靠性投递的机制

​	数据源必须是同一个

​	利用 Spring DataSourceTransactionManager，在本地事务提交的时候进行发送消息 

![UTOOLS1564550875894.png](https://i.loli.net/2019/07/31/5d4126dc560a692268.png)

### 消息幂等性

#### ​	可能非幂等的原因

​		可靠性消息投递机制

​		MQ Broker 服务与消费端传输消息的过程中的网络波动

​		消费端故障或异常

![UTOOLS1564550997673.png](https://i.loli.net/2019/07/31/5d4127562803460358.png)

#### 