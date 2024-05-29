[TOC]

# 楼兰 - RocketMQ高性能核心原理与源码架构剖析

## 1. 源码环境搭建

### 1. 主要功能模块

[官方Git仓库](https://github.com/apache/rocketmq)

[官方指定版本源码](http://rocketmq.apache.org/dowloading/releases/)

源码中几个最重要的模块：

- broker：broker启动进程
- client：消息客户端，producer/consumer 相关类
- example：示例代码
- namesrv：NameServer模块
- store：消息存储模块
- remoting：远程访问模块

### 2. 源码启动服务

```bash
mvn clean install -Dmaven.test.skip=true
```

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCEdc5e536e2a3feffba980580efc0ecd7e?ynotemdtimestamp=1715222167828)

编译完成后，进入调试阶段。

先在项目目录下创建一个conf目录，并从`distribution`拷贝`broker.conf`和`logback_broker.xml`和`logback_namesrv.xml`

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE57a4b014426ac2c4da663a897b25eae3?ynotemdtimestamp=1715222167828)

#### 2.1 启动nameServer

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE21a1642c9866586844d035d1fdcd81f5?ynotemdtimestamp=1715222167828)

启动报错，需要 `ROCKETMQ_HOME` 环境变量

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE9ff1a9ebd9fd0f3cd5f30d724d422114?ynotemdtimestamp=1715222167828)

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCEcaa8284876da1faff23caaaa2d240fc6?ynotemdtimestamp=1715222167828)

配置完成再次启动，看到如下内容表示启动成功

```apl
The Name Server boot success. serializeType=JSON
```

#### 2.2 启动Broker

启动之前，先修改之前复制的broker.conf文件

```apl
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH

# 自动创建Topic
autoCreateTopicEnable=true
# nameServ地址
namesrvAddr=127.0.0.1:9876
# 存储路径
storePathRootDir=E:\\RocketMQ\\data\\rocketmq\\dataDir
# commitLog路径
storePathCommitLog=E:\\RocketMQ\\data\\rocketmq\\dataDir\\commitlog
# 消息队列存储路径
storePathConsumeQueue=E:\\RocketMQ\\data\\rocketmq\\dataDir\\consumequeue
# 消息索引存储路径
storePathIndex=E:\\RocketMQ\\data\\rocketmq\\dataDir\\index
# checkpoint文件路径
storeCheckpoint=E:\\RocketMQ\\data\\rocketmq\\dataDir\\checkpoint
# abort文件存储路径
abortFile=E:\\RocketMQ\\data\\rocketmq\\dataDir\\abort
```

broker启动类是`BrokerStartup`,同样需要`ROCKETMQ_HOME`环境变量，还需要配置一个-c参数，指向broker.conf配置文件

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE3781ddaf0ab4b8dbde9fd4892f9b5651?ynotemdtimestamp=1715222167828)

#### 2.3 发送消息

在源码example模块下，启动 `org.apache.rocketmq.example.quickstart.Producer`

在源码中，需要指定NameServer地址

1. 配置一个 `NAMESRV_ADDR` 的环境变量

2. 在源码中加一行代码指定NameServer

   - ```java
     producer.setNamesrvAddr("127.0.0.1:9876");
     ```

#### 2.4 消费消息

在源码example模块下，启动 `org.apache.rocketmq.example.quickstart.Consumer`

运行时同样需要指定NameServer地址

```java
consumer.setNamesrvAddr("192.168.232.128:9876");
```

### 3. 读源码的方法

1. 带着问题读源码
2. 不要觉得一两遍就能读懂源码
3. 带上自己的理解，及时总结
4. 对各种扩展功能，尝试验证，试着理解源码中的各种单元测试

## 2. 源码热身阶段

### 1. NameServer的启动过程

#### 1. 关注重点

RocketMQ集群中，实际进行`消息存储、推动`等核心功能的是Broker

NameServer类似注册中心，只提供Broker端的服务注册和发现功能

#### 2. 源码重点

NameServer的启动入口类是`org.apache.rocketmq.namesrv.NamesrvStartup`

核心是构建并启动一个`NamesrvController`，类似MVC中的Controller。都是响应客户端请求，不过它是响应基于Netty的客户端请求

实际启动过程，可以配合NameServer的启动脚本进行更深入的理解

从NameServer启动和关闭这两个关键步骤，可以总结出其组件并不多，整体结构如下：

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE29c90d1551df6375a5585ec427ac47b3?ynotemdtimestamp=1715222167828) 

从这看出，RocketMQ的整体源码风格就是典型的MVC思想。Controller响应请求，Service处理业务，各种Table保存消息。

### 2. Broker服务启动过程

#### 1. 关注重点

Broker是整个RocketMQ的业务核心。负责所有消息存储、转发这些重要的业务

重点梳理Broker有哪些内部服务，这将是整理Broker核心业务流程的起点

#### 2. 源码重点

broker启动的入口在BrokerStartup，从它的main方法开始调试

启动过程关键点：围绕一个`BrokerController`对象，先创建，然后再启动

**首先：**在BrokerStartup.createBrokerController()中可以看到Broker的几个核心配置

- BrokerConfig：Broker服务配置
- MessageStoreConfig：消息存储配置
- NettyServerConfig：Netty服务端占用了10911端口
- NettyClientConfig：Broker既要作为Netty服务端，向客户端提供核心业务能力，又要作为Netty客户端，向NameServer注册心跳

**然后：**在BrokerController.start()可以看到启动了一大堆Broker的核心服务，挑一些重要的如下：

```java
this.messageStore.start();//启动核心的消息存储组件

this.remotingServer.start();
this.fastRemotingServer.start(); //启动两个Netty服务

this.brokerOuterAPI.start();//启动客户端，往外发请求

BrokerController.this.registerBrokerAll： //向NameServer注册心跳。

this.brokerStatsManager.start();
this.brokerFastFailure.start();//这也是一些负责具体业务的功能组件
```

Broker的一个整体结构如下：

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE43d60a879b526bdfc6203291131f1eb2?ynotemdtimestamp=1715227337088)

Broker启动了两个Netty服务，功能基本差不多。

在实际应用中，可以通过`producer.setSendMessageWithVIPChannel(true)、consumer.setVipChannelEnabled(true)`，让少量比较重要的msg走VIP通道。

## 3. 小试牛刀阶段

### 1. Netty服务注册框架

#### 1. 关注重点

网络通信服务是构建分布式应用的基础，也是理解RocketMQ底层业务的基础

重点梳理RocketMQ这个服务注册框架，理解各个业务进程之间是如何进行RPC通信的



Netty所有远程通信功能都由**remoting**模块实现。

RemotingServer模块里包含了RPC的服务端`RemotingServer`以及客户端`RemotingClient`

在RocketMQ中：

- NameServer主要是RPC的服务端
- Broker对于客户端来说，是RPC的服务端，而对NameServer来说，又是RPC的客户端
- 各种Client是RPC的客户端



RocketMQ基于Netty保持客户端与服务端的长连接Channel

只要Channel稳定，客户端和服务端就可以互相通信

例如：事务场景中，就需要broker多次主动向producer发送请求确认事务状态

所以，RemotingServer和RemotingClient都需要注册自己的服务

#### 2. 源码重点

1. 哪些组件需要Netty服务端？哪些组件需要Netty客户端？
   - NameServer需要NettyServer(`不需要NettyClient，验证NameServer之间是不需要进行数据同步`)
   - producer和consumer需要NettyClient
   - broker需要NettyServer响应客户端请求，需要NettyClient向NameServer注册心跳
   - 问题：事务消息的producer需要响应broker的事务状态回查，它需要NettyServer吗？

2. 所有的RPC请求数据都封装成`RemotingCommand`对象。而每个处理消息的服务逻辑，都会封装成一个`NettyRequestProcessor`对象
3. server和client都维护一个`processorTable`，这是个HashMap。
   1. key：服务码requestCode
   2. value：对应的运行单元 Pair<NettyRequestProcessor, ExecutorService>类型，包含了处理Processor和执行线程的线程池。
   3. 具体的Processor，由业务系统自行注册。
   4. Broker服务注册见：`BrokerController.registerProcessor()`，客户端服务注册见：`MQClientAPIImpl`
   5. NameServer会注册一个大的`DefaultRequestProcessor`,统一处理所有服务
4. 请求类型分`Request和Response`，这是为了支持异步的RPC调用
   1. NettyServer处理完请求后，可以先缓存到`responseTable`中，等NettyClient下次来获取，这样就不用阻塞Channel了，提升请求吞吐量。（Producer的同步请求流程是什么样的？）
5. 重点理解`remoting`包中是如何实现全流程异步化

整体RPC框架流程如下图：

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCEd94e49e13075ac5b40b6e552d7f38a74?ynotemdtimestamp=1715227337088)

RocketMQ使用Netty框架提供了一套基于服务码的服务注册机制，让各种不同组件都可以按需注册服务方法。

这套服务注册机制，非常简洁实用，在使用Netty进行其他相关应用开发时，都可以借鉴。

#### 3. 关于RocketMQ的同步结果推送与异步结果推送

remotingServer会维护一个responseTable，一个线程同步的Map结构。

key：请求id，value：异步的消息结果

```java
ConcurrentMap<Integer /* opaque */, ResponseFuture>
```

处理同步请求(NettyRemotingAbstract # invokeSyncImpl)时，处理结果存入responseTable

通过ResponseFuture提供一定的服务端异步处理支持，提升服务端的吞吐量

请求返回后，立即从responseTable移除该请求记录

**实际上，同步也是通过异步实现的**

```java
//org.apache.rocketmq.remoting.netty.ResponseFuture
 //发送消息后，通过countDownLatch阻塞当前线程，造成同步等待的效果。
    public RemotingCommand waitResponse(final long timeoutMillis) throws InterruptedException {
        this.countDownLatch.await(timeoutMillis, TimeUnit.MILLISECONDS);
        return this.responseCommand;
    }
 //等待异步获取到消息后，再通过countDownLatch释放当前线程。
    public void putResponse(final RemotingCommand responseCommand) {
        this.responseCommand = responseCommand;
        this.countDownLatch.countDown();
    }
```

处理异步请求(NettyRemotingAbstract # invokeAsyncImpl)时，处理的结果依然会存入responseTable，等待客户端后续再来请求结果。

保存的依然是一个ResponseFuture，即客户端请求时再去获取真正的结果

另外，在RemotingServer启动时，会启动一个定时线程任务，不断扫描responseTable，将过期的response清除

```java
//org.apache.rocketmq.remoting.netty.NettyRemotingServer
this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    NettyRemotingServer.this.scanResponseTable();
                } catch (Throwable e) {
                    log.error("scanResponseTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
```

### 2. Broker心跳注册管理

#### 1. 关注重点

broker在启动时向所有NameServer注册自己的服务信息，并定时往NameServer发送心跳信息

NameServer会维护Broker的路由列表，并对路由表进行实时更新

#### 2. 源码重点

启动注册心跳：`BrokerController.this.registerBrokerAll`。然后启动一个定时任务，以10s延迟，默认30s的间隔持续向NameServer发送心跳

NameServer内部会通过RouteInfoManager及时维护Broker信息，在启动时，会启动定时任务，扫描不活动的Broker。方法入口：`NamesrvController.initialize()`

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE9cd19e3fe68a4e4f6c65f60e9a9af628?ynotemdtimestamp=1715227337088)

#### 3. 极简化的服务注册实现流程

RocketMQ为什么不用zk、nacos之类的，而是自己搞一个NameServer？

首先：依赖外部组件会影响产品的独立性，不利于版本演进。Kafka要抛弃zk就是这样。

另外：NameServer之间不进行信息同步，而是依赖broker向所有NameServer同时发起注册。这样NameServer非常轻量，而zk、nacos有一堆其他功能。

注意：这种极简设计，是牺牲数据一致性为代价。Broker注册时，可能部分NameServer注册失败，多个NameServer之间就数据不同步了。作为大的注册中心来讲，不可接受。但对RocketMQ，client只要从NameServer获取一个正常运行的broker即可，并不需要完整的broker列表。

### 3. Producer发送消息过程

#### 1. 关注重点

Producer有两种：

1. 普通发送者：DefaultMQProducer，只负责发送消息。
2. 事务消息发送者：TransactionMQProducer，支持事务消息机制。
   - 需要在事务消息过程中，提供事务状态确认的服务，这就要求事务消息发送者虽然是一个client，但得完成整个事务消息的确认机制后才能退出。

目前只关注DefaultMQProducer的消息发送过程



整个Producer的使用流程，大致分为两个步骤：

1. 调用start()，进行一大堆准备工作
2. 各种send()，进行消息发送



重点关注以下几个问题：

1. Producer启动过程中启动了哪些服务？
2. Producer如何管理broker路由信息？（producer启动后，NameServer挂了，producer还能发消息吗？）
3. 关于Producer的负载均衡，即producer到底将msg发到哪个MessageQueue中。可以结合顺序消息机制来理解，消息中的MessageSelector到底是如何工作的。

#### 2. 源码重点

##### 1. producer的启动流程

所有producer启动，最终都会调用到`DefaultMQProducerImpl # start()`。

在start()中通过一个MqClientFactory对象，启动producer的一大堆重要服务

> 这里是一种设计模式，虽然有很多种不同的client，但它们的启动流程最终都是统一的，全是交由MqClientFactory对象来启动。
>
> 不同之处在于这些客户端在启动过程中，按照服务端的要求注册不同的信息。
>
> 例如：producer注册到producerTemplate，consumer注册到consumerTable，管理控制端注册到adminExtTable

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE949f118118049b9da7afe3b109ac9d1b?ynotemdtimestamp=1715227337088)

##### 2. 发送消息的核心流程

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCEe17b1d90bd0cf558b3b24f40d7d4dba8?ynotemdtimestamp=1715227337088)

1. 发送消息时，会维护一个本地的`topicPublishInfoTable`缓存，DefaultMQProducer会尽量保证这个缓存数据是最新的。

   - 如果NameServer挂了，DefaultMQProducer还是会基于这个本地缓存区找Broker
   - 只要能找到Broker，还是可以正常发消息

2. producer如何找MessageQueue：

   - 默认情况下，producer轮询各个MessageQueue

   - 但如果某一次往一个broker发送请求失败后，下次就会跳过这个broker

   - ```java
     //org.apache.rocketmq.client.impl.producer.TopicPublishInfo
      //如果进到这里lastBrokerName不为空，那么表示上一次向这个Broker发送消息是失败的，这时就尽量不要再往这个Broker发送消息了。
      public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
          if (lastBrokerName == null) {
              return selectOneMessageQueue();
          } else {
              for (int i = 0; i < this.messageQueueList.size(); i++) {
                  int index = this.sendWhichQueue.incrementAndGet();
                  int pos = Math.abs(index) % this.messageQueueList.size();
                  if (pos < 0)
                      pos = 0;
                  MessageQueue mq = this.messageQueueList.get(pos);
                  if (!mq.getBrokerName().equals(lastBrokerName)) {
                      return mq;
                  }
              }
              return selectOneMessageQueue();
          }
      }
     ```

3. 如果发消息时，传了Selector，producer就不会再走负载的逻辑，而是用Selector去找一个队列

   - 参见 `DefaultMQProducerImpl # sendSelectImpl()`

### 4. Consumer拉取消息过程

#### 1. 关注重点

- consumer也有两种：pull/push。`MQ高级目标：提升整个消息处理的性能。服务端的优化手段往往不够直接，最直接的就是对consumer进行优化`
  - RocketMQ中，consumer的业务逻辑最复杂，这里重点关注用得最多的push consumer
- consumer group之间有`集群模式和广播模式`，了解这两种模式的逻辑封装
- 关注consumer的负载均衡原理(`consumer如何绑定消费队列，消费策略如何落地`)
- push consumer中，`MessageListenerConcurrently和MessageListenerOrderly`这两种消息监听器的处理逻辑到底有何不同，后者为什么能保证消息顺序

#### 2. 源码重点

consumer核心启动过程和producer是一样的，最终都是通过MQClientFactory对象启动，不过添加了一些注册信息

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCEca0869432564877333e954b3ddaeb5c2?ynotemdtimestamp=1715227337088)

#### 3. 广播模式与集群模式的Offset处理

在`DefaultMQPushConsumerImpl # start()` 中，启动了非常多的核心服务。

比如：对于广播模式与集群模式的Offset处理

```java
 if (this.defaultMQPushConsumer.getOffsetStore() != null) {
   this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
 } else {
   switch (this.defaultMQPushConsumer.getMessageModel()) {
     case BROADCASTING:
       this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
       break;
     case CLUSTERING:
       this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
       break;
     default:
       break;
   }
   this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
 }
this.offsetStore.load();
```

广播模式使用`LocalFileOffsetStore`，在consumer本地保存offset

集群模式使用`RemoteBrokerOffsetStore`，在broker端保存offset

这两种offset的存储方式，最终都是通过维护本地的`offsetTable`缓存来管理offset

#### 4. consumer与MessageQueue建立绑定关系

start()中给`rebalanceImpl`设定了一个`AllocateMessageQueueStrategy`，用来给consumer分配MessageQueue

```java
this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
//Consumer负载均衡策略
this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
```

`AllocateMessageQueueStrategy` 就是用来给consumer和MessageQueue之间建立一种对应关系的。

即：只要Topic当中的MessageQueue以及同一个ConsumerGroup中的consumer实例都没有变动，

那么，某个consumer实例只是消费固定的一个或多个MessageQueue上的消息。

其他consumer不会来抢这个consumer对应的MessageQueue（`broker需要按照consumerGroup管理每个MessageQueue上的offset，如果一个MessageQueue上有多个同属一个consumerGroup的consumer，它们处理进度就会不一样，offset就乱套了`）

#### 5. 顺序消费与并发消费

start()中，启动了`consumerMessageService`线程，进行消息拉取

```java
//Consumer中自行指定的回调函数。
if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
  this.consumeOrderly = true;
  this.consumeMessageService =
    new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
  this.consumeOrderly = false;
  this.consumeMessageService =
    new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}
```

consumer通过`registerMessageListener()`指定的回调函数，都被封装成了`ConsumerMessageService的子实现类`

对这两个服务实现类的调用，会延续到`DefaultMQPushConsumerImpl的pullCallback对象中`，即consumer每拉进来一批消息后，就向broker提交下一个拉取消息的请求

> 验证点：顺序消息只对异步消费(推模式)有效。
>
> 同步消费的拉模式是无法进行顺序消费的。因为这个pullCallback对象，在拉模式的同步消费时，根本就没有往下传。
>
> 当然，这并不是说拉模式不能锁定队列进行顺序消费，拉模式在consumer端应用就可以指定从哪个队列上拿消息。

```java
PullCallback pullCallback = new PullCallback() {
  @Override
  public void onSuccess(PullResult pullResult) {
    if (pullResult != null) {
      //...
      switch (pullResult.getPullStatus()) {
        case FOUND:
          //...
          DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
            pullResult.getMsgFoundList(),
            processQueue,
            pullRequest.getMessageQueue(),
            dispatchToConsume);

          //... 
          break;
          //...
      }
    }
  }
```

这里提交的，实际上是一个`ConsumeRequest线程`，而这个线程，在两个不同的ConsumerService中有不同的实现

其中，两者最为核心的区别在于`ConsumerMessageOrderlyService ` 是锁定了一个队列，处理完了之后，再消费下一个队列

```java
 public void run() {
   // ....
   final Object objLock = messageQueueLock.fetchLockObject(this.messageQueue);
   synchronized (objLock) {
     //....
   }
 }
```

为什么给队列加个锁，就能保证顺序消费？结合顺序消息的实现机制理解下：

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE644c04092e89bd526ad08b78e5aa8b9b?ynotemdtimestamp=1715227337088)

源码看出，consumer提交请求时，都是往线程池里异步提交的请求。

如果不加队列锁，那么就算consumer提交针对同一个MessageQueue的拉取消息请求，这些请求也都是异步执行，返回顺序就是乱的。

给队列加锁后，就保证了串行操作，这就是异步情况下保证顺序的基础思路。

#### 6. 实际拉取消息还是通过PullMessageService完成的

start()中，相当于对很多consumer服务进行初始化，包括指定一些服务的实现类，以及启动一些定时的任务线程，比如：清理过期的请求缓存等。

最后，会随着MQClientFactory组件的启动，启动一个PullMessageService。

实际的消息拉取都交由`PullMessageService`进行

所谓消息推模式，其实还是通过Consumer拉模式实现的

```java
 //org.apache.rocketmq.client.impl.consumer.PullMessageService
 private void pullMessage(final PullRequest pullRequest) {
   final MQConsumerInner consumer = this.mQClientFactory.selectConsumer(pullRequest.getConsumerGroup());
   if (consumer != null) {
     DefaultMQPushConsumerImpl impl = (DefaultMQPushConsumerImpl) consumer;
     impl.pullMessage(pullRequest);
   } else {
     log.warn("No matched consumer for the PullRequest {}, drop it", pullRequest);
   }
 }
```

### 5. 客户端负载均衡管理总结

从之前producer发消息的过程、consumer拉取消息的过程，可以抽象出一个RocketMQ消息分配的管理模型

这个模型是使用RocketMQ时，很重要的进行性能优化的依据。

#### 1. Producer负载均衡

producer发送msg时，默认轮询目标Topic下的所有MessageQueue

并采用`递增取模`的方式往不同的MessageQueue上发消息，已达到平均分配在不同queue

由于MessageQueue是分布在不同的Broker上的，所以消息也会发送到不同的Broker上



之前源码看到，如果往某个broker发msg失败了，下次就会尽量避免往该broker发msg

但如果应用场景允许发送消息长延迟，也可以给product设定`setSendLatencyFaultEnable(true)`

这对某些broker集群的网络不是很好的环境，可以提高消息发送成功率

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE94e4d57101ac23117eff7505bd588cf3?ynotemdtimestamp=1715227337088)

producer在发msg时，可以指定一个MessageQueueSelector，将msg落到指定MessageQueue上

这可以保证消息局部有序

#### 2. Consumer负载均衡

consumer也是以MessageQueue为单位来进行负载均衡。分为广播模式和集群模式。

##### 1. 集群模式

msg只需投递到订阅这个topic的consumer group的任意实例即可。

rocketmq采用主动拉取的方式拉取并消费消息，在拉取时需要明确指定拉取哪一条messageQueue



每当实例数量发生变更，都会触发一次所有实例的负载均衡，这时会按照queue的数量和实例的数量平均分配queue给每个实例

每次分配时，都会将MessageQueue和消费者id进行排序后，再用不同的分配算法进行分配

内置的分配算法共有6种，分别对应`AllocateMessageQueueStrategy`下的6种实现类

可以在consuemr中直接set来指定，默认使用最简单的 `平均分配策略`

- `AllocateMachineRoomNearby：`将同机房的consumer和broker优先分配在一起
- `AllocateMessageQueueAveragely：`平均分配。将所有MessageQueue平均分给每一个消费者
- `AllocateMessageQueueAveragelyByCircle：`轮询分配。轮流给每个消费者分配一个MessageQueue
- `AllocateMessageQueueByConfig：`不分配，直接指定一个messageQueue列表。类似于广播模式，直接指定所有队列
- `AllocateMessageQueueByMachineRoom：`按逻辑机房的概念进行分配。又是对BrokerName合ConsumerIdc有定制化的配置
- `AllocateMessageQueueConsistentHash：`这个一致性哈希策略只需要指定一个虚拟节点数，是用的一个哈希环的算法，虚拟节点是为了让Hash数据在环上分布更为均匀。

最常用的就是平均分配和轮询分配了，例如：平均分配时的情况如下

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE45a321b92ac66306eb8700a72da7ad20?ynotemdtimestamp=1715227337088)

而轮询分配就不计算了，每次把一个队列分给下一个Consumer实例

##### 2. 广播模式

每条msg都会投递给订阅了topic的所有consumer实例，也就没有消息分配这一说。

而在实现上，就是在consumer分配queue时，所有consumer都分到所有的queue

实现关键：`将consumer的offset不再保存到broker，而是保存在consumer所在的client中，由其自行维护`

## 4. 融汇贯通阶段

### 1. RocketMQ的持久化文件结构

消息持久化：将内存中的消息写入到本地磁盘的过程

而磁盘IO通常是一个很耗性能的操作，所以，这是一个MQ产品提升性能的关键(甚至最为重要)



进入源码前，首先看RocketMQ在磁盘存了哪些文件。msg默认路径`${user_home}/store`，这些存储目录可以在broker.conf中自行指定

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE9a3b326e79ebf7a51daa0b00c06dc98c?ynotemdtimestamp=1715227337088)

存储文件主要分为三个部分：

- `commitLog`：存储消息的元数据。所有消息都会顺序存入到commitLog文件当中。
  - commitLog由多个文件组成，每个文件固定大小1G。以第一条消息的偏移量为文件名
- `consumerQueue`：存储消息在commitLog的索引。一个MessageQueue一个文件
  - 记录当前MessageQueue被哪些消费者组消费到了哪一条commitLog
- `IndexFile`：为了消息查询提供了一种通过key或时间区间来查询消息的方法
  - 通过indexFile来查找消息，不影响发送与消费消息的主流程

还有几个辅助的存储文件，主要记录一些描述信息的元数据

- checkpoint：数据存盘检查点。
  - 主要记录`commitlog、consumequeue、indexfile`文件最后一次刷盘的时间戳
- config/*.json：存储RocketMQ的一些关键配置信息
  - 例如：Topic配置、消费者组配置、消费者组消息偏移量 等
- abort：RocketMQ用来判断程序是否正常关闭的一个标识文件。
  - 正常情况下，启动时创建，关闭时删除
  - 但遇到服务宕机、kill -9等非正常关闭服务的情况，这个abort文件就不会删除，后续就会做一些数据恢复的操作



整体的消息存储结构，官方做了个图进行描述：

![](https://note.youdao.com/yws/public/resource/4cda837bfb79db5ddb5e26e46ee5a12c/WEBRESOURCE1174b1eb33288f8ca76567ff11435fb4?ynotemdtimestamp=1715227337088)

简单来说，producer发送的所有msg，不管属于哪个topic，broker都统一存在commitlog中

然后分别构建consumerQueue和indexFile 两个索引文件，用于辅助消费者进行消息检索

这种设计最直接的好处就是可以减少查找目标文件的时间，让消息以最快的速度落盘

对比Kafka存文件，需要寻找消息所属的Partition文件，再完成写入，当Topic较多时，这样的Partition寻址就会浪费非常多的时间。



在文件形式上：

commitlog文件大小固定，文件名就是存储的第一条msg的offset



consumerQueue文件主要是加速consumer进行消息索引。

- 每个文件夹对应RocketMQ中的一个MessageQueue
- 文件夹下的文件记录了每个MessageQueue中的消息在CommitLog文件当中的偏移量。
- 这样，consumer就可以凭此快速找到msg
- 而consumer在consumerQueue中的消费进度，会保存在config/consumerOffset.json文件当中



indexFile文件主要辅助消费者进行消息索引。

- 如果消费者指定时间戳进行消费，或按照MessageId、MessageKey来检索文件，比如控制台的消息轨迹功能，consumerQueue文件就不够用了，indecFile就是用来辅助这类消息检索的。
- 它文件名比较特殊，不是以offset命名，而是用时间命名。
- 也是一个固定大小的文件



> 以上是对RocketMQ存盘文件最基础的了解，但只有这样的设计，还支撑不了三高性能。
>
> RocketMQ如何保证ConsumerQueue、IndexFile两个索引文件与CommitLog中的消息对齐？
>
> 如何保证消息断电不丢失？
>
> 如果保证文件高效的写入磁盘？
>
> 想要了解三高问题的核心设计，还是得深入源码

### 2. commitLog写入

消息存储入口：`DefaultMessageStore.asyncPutMessage()`，在broker处理producer发消息的请求`SendMessageProcessor`中

CommitLog的`asyncPutMessage`方法中，会给写入线程加锁，保证一次只会允许一个线程写入

最终进入CommitLog的`DefaultAppendMessageCallback # doAppend()` 

- 这就是Broker写入消息的实际入口。最终会把消息追加到`MappedFile`映射的一块内存里，并没有直接写入磁盘
- 在随后调用`CommitLog # submitFlushRequest()`，提交刷盘申请，完成后，内存中的消息才会真正写入到磁盘
- 提交刷盘申请后，立即调用`CommitLog # submitReplicaRequest()` 发起主从同步申请

### 3. 文件同步刷盘与异步刷盘

入口：`CommitLog # submmitFlushRequest()`

这里涉及到对于同步刷盘和异步刷盘的不同处理机制。里面有很多极致提高性能的设计，对理解和设计高并发应用场景有非常大的借鉴意义。

同步或异步，是通过不同的`FlushCommitLogService`的子服务实现的

```java
 //org.apache.rocketmq.store.CommitLog的构造方法
  if (FlushDiskType.SYNC_FLUSH == defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
    this.flushCommitLogService = new GroupCommitService();
  } else {
    this.flushCommitLogService = new FlushRealTimeService();
  }

this.commitLogService = new CommitRealTimeService();
```

同步刷盘：采用 `GoupCommitService` 子线程

- 虽然叫同步，但从源码看到，实际是每10ms执行一次doCommit()
- 扫描文件缓存，只要缓存当中有消息，就执行一次flush操作

异步刷盘：采用 `FlushRealTimeService` 子线程

- 最终也是执行flush操作，但执行时机会根据配置灵活调整



异步刷盘和同步刷盘的最本质区别：`进行flush操作的频率不同`

> 常说的RocketMQ同步刷盘，可以保证Broker断电消息不丢失，其实是有失偏颇的。
>
> 海量数据下，来一条msg就刷一次盘，操作系统是承受不了的
>
> 所以，对于消息安全性的设计，其实是重在取舍，无法做到绝对



同步刷盘和异步刷盘最终落地到`FileChannel # force()`

- 此方法最终调用一次操作系统的`fsync`系统调用，完成文件写入
- 关于force操作的详细演示，参考后面的零拷贝部分。

```java
//org.apache.rocketmq.store
 public int flush(final int flushLeastPages) {
   if (this.isAbleToFlush(flushLeastPages)) {
     if (this.hold()) {
       int value = getReadPosition();

       try {
         //We only append data to fileChannel or mappedByteBuffer, never both.
         if (writeBuffer != null || this.fileChannel.position() != 0) {
           this.fileChannel.force(false);
         } else {
           this.mappedByteBuffer.force();
         }
       } catch (Throwable e) {
         log.error("Error occurred when force data to disk.", e);
       }

       this.flushedPosition.set(value);
       this.release();
     } else {
       log.warn("in flush, hold failed, flush offset = " + this.flushedPosition.get());
       this.flushedPosition.set(getReadPosition());
     }
   }
   return this.getFlushedPosition();
 }
```



另外一个`CommitRealTimeService`子线程，是用来写入堆外内存的。