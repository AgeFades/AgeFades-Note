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

##### 			MessageConverter : 消息转换器

​				Jackson2JsonMessageConverter : Json

​				DefaultJackson2JavaTypeMapper : Java 对象

​				

##### 