[TOC]

# 楼兰 - RocketMQ核心编程模型以及生产环境最佳实践

## 1. 深入理解RocketMQ消息模型

### 1. RocketMQ客户端基本流程

```xml
<!-- 核心依赖 -->
<dependency>
        <groupId>org.apache.rocketmq</groupId>
        <artifactId>rocketmq-client</artifactId>
        <version>4.9.5</version>
</dependency>
<!-- 权限控制相关 -->
<dependency>
        <groupId>org.apache.rocketmq</groupId>
        <artifactId>rocketmq-acl</artifactId>
        <version>4.9.5</version>
</dependency>
```

```java
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        //初始化一个消息生产者
        DefaultMQProducer producer = new DefaultMQProducer("please_rename_unique_group_name");
        // 指定nameserver地址
        producer.setNamesrvAddr("192.168.232.128:9876");
        // 启动消息生产者服务
        producer.start();
        for (int i = 0; i < 2; i++) {
            try {
                // 创建消息。消息由Topic,Tag和body三个属性组成，其中Body就是消息内容
                Message msg = new Message("TopicTest","TagA",("Hello RocketMQ " +i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                //发送消息，获取发送结果
                SendResult sendResult = producer.send(msg);
                System.out.printf("%s%n", sendResult);
            } catch (Exception e) {
                e.printStackTrace();
                Thread.sleep(1000);
            }
        }
        //消息发送完后，停止消息生产者服务。
        producer.shutdown();
    }
}
```

```java
public class Consumer {
    public static void main(String[] args) throws InterruptedException, MQClientException {
        //构建一个消息消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name_4");
        //指定nameserver地址
       consumer.setNamesrvAddr("192.168.232.128:9876");
       consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        // 订阅一个感兴趣的话题，这个话题需要与消息的topic一致
        consumer.subscribe("TopicTest", "*");
        // 注册一个消息回调函数，消费到消息后就会触发回调。
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,ConsumeConcurrentlyContext context) {
    msgs.forEach(messageExt -> {
                    try {
                        System.out.println("收到消息:"+new String(messageExt.getBody(), RemotingHelper.DEFAULT_CHARSET));
                    } catch (UnsupportedEncodingException e) {}
                });
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        //启动消费者服务
        consumer.start();
        System.out.print("Consumer Started");
    }
}
```

**消息生产者的固定步骤**

1. 创建producer，并指定生产者组名
2. 指定Nameserver地址
3. 启动producer
4. 创建消息对象，指定Topic、Tag、Body
5. 发送消息
6. 关闭Producer

**消息消费者的固定步骤**

1. 创建consumer，必须指定消费者组名
2. 指定Nameserver地址
3. 订阅Topic、Tag
4. 设置回调函数，处理消息
5. 启动consumer，消费者会一直挂起，持续处理消息

指定NameServer的方式有两种。可以直接在客户端指定（优先级更高），或者通过读系统环境变量

### 2. 消息确认机制

消息安全：

1. producer 确保发送至 broker
2. consumer 确保能从 broker 消费消息

#### 1. producer采用消息确认+多次重试

##### 单向发送

producer只管发，不管broker收没收到

```java
public class OnewayProducer {
    public static void main(String[] args)throws Exception{
        DefaultMQProducer producer = new DefaultMQProducer("producerGroup");
        producer.start();
        Message message = new Message("Order","tag","order info : orderId = xxx".getBytes(StandardCharsets.UTF_8));
        producer.sendOneway(message);
        Thread.sleep(50000);
        producer.shutdown();
    }
}
```

适用于要求效率，允许消息丢失的场景，比如日志

##### 同步发送

producer发消息后，阻塞等待broker响应结果

```java
 SendResult sendResult = producer.send(msg);
 
 // sendResult中的sendStatus属性
 public enum SendStatus {
    SEND_OK,
    FLUSH_DISK_TIMEOUT,
    FLUSH_SLAVE_TIMEOUT,
    SLAVE_NOT_AVAILABLE,
}
```

send_ok表示成功，其他几种枚举都表示失败，失败时采取补救机制，比如重试、入库后续人工补发等

非send_ok时，并不一定表示broker没有把消息推给consumer。所以producer、consumer要约定唯一业务id，保证幂等

效率低，适用于数据安全性要求高的场景

##### 异步发送

producer发消息时，注册一个回调函数，可以做别的事，等broker端响应后会触发这个回调函数的逻辑

```java
producer.send(msg, new SendCallback() {
  @Override
  public void onSuccess(SendResult sendResult) {
    countDownLatch.countDown();
    System.out.printf("%-10d OK %s %n", index, sendResult.getMsgId());
  }
  @Override
  public void onException(Throwable e) {
    countDownLatch.countDown();
    System.out.printf("%-10d Exception %s %n", index, e);
    e.printStackTrace();
  }
});
```

broker响应太慢，也会触发onException，超时时间默认3s，可以通过 **producer.setSendMsgTimeout** 修改

回调函数被触发之前，producer不能调用shutdown()，不然主线程结束，回调函数也不会被触发了

较好的兼容信息安全性、producer高吞吐要求，但也不是万能的，对主线程有侵入性，具体场景具体考虑

#### 2. consumer采用状态确认机制

核心思路都是等接收端确认，这边是broker等consumer返回状态

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
```

consumer返回CONSUME_SUCCESS，就表示消费成功

当返回RECONSUME_LATER时，表示消费失败，broker过一段时间再发起消息重试推送

> RocketMQ为兼容重试机制的成功率和性能，设计了一套非常完善的消息重试机制

1. broker推送失败重试16次后，会把这个消息推到消费者组对应的死信topic，等人工处理或者彻底删掉这些消息，默认最大重试次数是16次
2. broker给每个消费者组设计对应的重试Topic，消除对原本MessageQueue的影响（FIFO，一直重试导致阻塞）
3. 消息重试时，broker会往消费者组中随意一个实例推送，但客户端的实现中，MQDefaultMQConsumer并没有强制规定消费者组不能重复，所以可能存在一些topic和消费逻辑完全不同的consumer，共同组成一个消费组，这时broker认为重试成功，但message的消费逻辑可能不一致。这是实际应用业务开发需要注意避免的地方
4. consumer尽量同步消费消息，不要开异步线程处理，不然可能消费失败，主线程又直接给broker消费成功的响应

#### 3. consumer自行指定起始消费位点

offset由broker维护，也允许consumer自行指定，来解决消费位点不一致的情况

因为很难精准知道消费位点，所以提供了一个枚举

```java
public enum ConsumeFromWhere {
    CONSUME_FROM_LAST_OFFSET, //从队列的第一条消息开始重新消费
    CONSUME_FROM_FIRST_OFFSET, //从上次消费到的地方开始继续消费
    CONSUME_FROM_TIMESTAMP; //从某一个时间点开始重新消费
}
```

如果指定 CONSUME_FROM_TIMESTAMP，需要通过Consumer的另一个属性ConsumerTimestamp传入一个表示时间的字符串

```apl
consumer.setConsumerTimestamp("20131223171201");
```

### 3. 广播消息

**应用场景**

集群模式：一个message只会被一个消费组中的多个consumer共同处理一次

广播模式：一个message会被推送给所有consumer处理，不再关心消费者组

**示例代码**

启动多个consumer，在广播模式下，这些consumer都会消费一次信息

```apl
consumer.setMessageModel(MessageModel.BROADCASTING);
```

**实现思路**

默认模式（集群模式）下，broker给每个consumerGroup维护一个统一的offset，保证一个消息在同一个consumerGroup内只会被消费一次

广播模式是将offset转移到consumer自行维护，broker就只管向所有consumer推送消息，而不用负责维护offset

**注意点**

1. broker不维护offset，意味着consumer处理消息失败，broker无法进行消息重试
2. consumer维护offset的作用是，服务重启不影响消息消费
   1. 比如：producer发1-10消息，consumer消费到6挂了，重启后，可以用自己的offset再消费6-10
   2. 但是consumer的offset丢失的话，可以正常消费后面的，但是6-10就消费不到了
   3. offset维护在 **${user.home}/.rocketmq_offset/${clientIp}${instanceName}/${group}/offsets.json**

consumer存储广播消息的offset文件默认目录是 **System.getProperty(“user.home”) + File.separator + “.rocketmq_offsets” **,可以通过 **rocketmq.client.localOffsetStoreDir** 系统属性进行修改



本地offsets文件在缓存目录中的具体位置，与consumer的clientIp和instanceName有关

其中instanceName默认是DEFAULT，可以通过 **rocketmq.client.name** 进行修改

另外，每个consumer都可以单独设置instanceName



RocketMQ会通过定时任务不断尝试本地offset文件的写入，但是写入失败的话，不会进行任何补救

### 4. 顺序消息机制

**应用场景**

每个订单步骤，下单、锁库存、下物流...每个业务步骤都由一个producer通知给MQTT，如何保证consumer处理顺序不乱？

**示例代码**

```java
for (int i = 0; i < 10; i++) {
  int orderId = i;
  for(int j = 0 ; j <= 5 ; j ++){
    Message msg =
      new Message("OrderTopicTest", "order_"+orderId, "KEY" + orderId,
                  ("order_"+orderId+" step " + j).getBytes(RemotingHelper.DEFAULT_CHARSET));
    SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
      @Override
      public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
      }
    }, orderId);
    System.out.printf("%s%n", sendResult);
  }
}

```

通过MessageSelector，将orderId相同的消息，都转发到同一个MessageQueue

consumer核心代码：

```java
consumer.registerMessageListener(new MessageListenerOrderly() {
  @Override
  public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
    context.setAutoCommit(true);
    for(MessageExt msg:msgs){
      System.out.println("收到消息内容 "+new String(msg.getBody()));
    }
    return ConsumeOrderlyStatus.SUCCESS;
  }
});
```

**实现思路**

只有放到一起的一批消息，才有可能保持消息的顺序

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEb71d36417c2dd8b405da1d86a67541cf?ynotemdtimestamp=1714272019227)

1. producer只有将一批有顺序要求的消息，放到同一个MessageQueue，Broker才可能保持这一批消息的顺序
2. consumer只有一次锁定一个MessageQueue，拿到MessageQueue上的所有消息

**注意点**

1. 理解局部有序与全局有序，大部分业务场景下，需要的其实是局部有序，如果要保持全局有序，那就只保留一个MessageQueue，性能显然会非常低
2. producer尽可能将有序消息打散到不同的MessageQueue上，避免过于集中导致数据热点竞争
3. consumer只能用同步的方式处理消息，不要使用异步处理，更不能自行使用批量处理
4. consumer只进行有限次数的重试，如果一条消息处理失败，RocketMQ会将后续消息阻塞住，让consumer进行重试。但是如果consumer一直失败，超过最大重试次数，RocketMQ就会跳过这一条消息，处理后面的消息，造成消息乱序
5. consumer如果确实处理逻辑出现问题，不建议抛出异常，可以返回 **SUSPEND_CURRENT_QUEUE_A_MOMENT** 

### 5. 延迟消息

**应用场景**

Message到Broker后，不是立即被consumer消费，而是延迟一定时间后

> RabbitMQ只能使用死信队列变相实现延迟消息，或者加装一个插件来支持延时消息
>
> Kafka不太好实现延迟消息

**实例代码**

producer核心代码

```apl
msg.setDelayTimeLevel(3);
```

RocketMQ给Message定制了18个默认的延迟级别，分别对应18个不同的预设好的延迟时间

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEd53a260089c0e9f65f44e5f63d22a275?ynotemdtimestamp=1714272019227)

**实现思路**

延时消息难点在不断进行定时轮询，影响性能。

扫全部消息显然不可能，RocketMQ的实现方式是预设一个系统级Topic(**SCHEDULE_TOPIC_XXXX**)

在该Topic下，预设18个延迟队列，每次只针对这些队列里的消息进行延迟操作，就不会影响全局

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEd3a041207ccc55167f835e8b60ae9825?ynotemdtimestamp=1714272019227)

**注意点**

5.x之前的延迟消息是不太灵活的，5.x版本之后支持预设一个具体的时间戳，按秒的精度进行定时发送

这18个延迟级别虽然无法调整，但是每个延迟级别对应的延迟时间是可以调整的，但通常不这么干

### 6. 批量消息

**应用场景**

生产者要发送的消息较多时，可以将多条消息合并成一条。减少网络IO，提升消息发送的吞吐量

**示例代码**

producer核心代码

```java
List<Message> messages = new ArrayList<>();
messages.add(new Message(topic, "Tag", "OrderID001", "Hello world 0".getBytes()));
messages.add(new Message(topic, "Tag", "OrderID002", "Hello world 1".getBytes()));
messages.add(new Message(topic, "Tag", "OrderID003", "Hello world 2".getBytes()));

producer.send(messages);
```

**注意点**

批量消息使用简单，但是同一批消息的Topic必须相同，另外，不支持延迟消息

批量消息的大小不要超过1M，超过需要自行分割

### 7. 过滤消息

**应用场景**

同一个Topic下有多种不同的消息，consumer只希望关注某一类消息

例：仓储系统一个Topic，该Topic下，存在入库、出库等不同类型的消息，那具体业务逻辑就需要过滤出自己关注的消息

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE87da9bbc25ee7639ba013470c24c7e80?ynotemdtimestamp=1714272019227)

**示例代码1：简单过滤**

producer发送消息时，增加Tag属性

```java
 String[] tags = new String[] {"TagA", "TagB", "TagC"};

for (int i = 0; i < 15; i++) {
  Message msg = new Message("TagFilterTest",
                            tags[i % tags.length],
                            "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
  SendResult sendResult = producer.send(msg);
  System.out.printf("%s%n", sendResult);
}
```

consumer通过Tag订阅

```apl
consumer.subscribe("TagFilterTest", "TagA");
```

**示例代码2：SQL过滤**

通过Tag属性，只能进行简单的消息匹配

要进行复杂的消息过滤，如数字比较、模糊匹配等，就需要使用SQL过滤方式

SQL过滤方式可以通过Tag属性以及用户自定义属性一起，以标准SQL的方式进行消息过滤

producer在发消息时，除了Tag属性外，还可以增加自定义属性。核心代码：

```java
String[] tags = new String[] {"TagA", "TagB", "TagC"};

for (int i = 0; i < 15; i++) {
  Message msg = new Message("SqlFilterTest",
                tags[i % tags.length],
                ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET)
            );
  msg.putUserProperty("a", String.valueOf(i));

  SendResult sendResult = producer.send(msg);
  System.out.printf("%s%n", sendResult);
}
```

consumer在进行过滤时，可以指定一个标准的SQL语句，定制复杂的过滤规则，核心代码：

```java
consumer.subscribe("SqlFilterTest",
            MessageSelector.bySql("(TAGS is not null and TAGS in ('TagA', 'TagB'))" +
                "and (a is not null and a between 0 and 3)"));
```

**实现思路**

Tag和用户自定义属性，都是随着消息一起传递的，consumer是可以拿到的，比如：

```java
consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.println(msg.getTags());
                    System.out.println(msg.getProperties());
                }
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
```

broker在往consumer推消息时，在broker端进行消息过滤。

tag属性就是直接匹配，sql是通过anltr引擎来解析再过滤的

> anltr是一个开源的SQL语句解析框架，很多开源产品都在使用ANLTR来解析SQL语句
>
> 比如：ShardingSphere，Flink...

**注意点**

1. 使用Tag过滤时，可以使用 **||** 连接多个Tag值，也可以用 ***** 匹配所有
2. 使用SQL过滤时，遵循SQL92标准，支持一些常见的基本操作
   - 数值比较：> >= < <= between =
   - 字符比较：= <> in
   - is null 或者 is not null
   - 逻辑符号：and or not
3. broker和consuemr都可以做消息过滤，broker过滤减少io，consumer过滤就是收所有消息，不感兴趣的就返回不成功，跳过
4. consumer不感兴趣的消息，通常需要在同一个消费组定制另外的消费者示例来消费。但一直没有对应consumer的话，broker还是会推进offset

### 8. 事务消息

**应用场景**

通过RocketMQ的事务机制，来保证上下游的数据一致性

**案例场景**

电商，支付订单操作，涉及 下游物流发货、积分变更、购物车状态清空等多个子系统的变更

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE27ad6dd6e1de14de0247dbb519e99987?ynotemdtimestamp=1714350864395)

这几个业务要一起成功或一起失败，用分布式事务也可以做，但是很麻烦

用RocketMQ与consumer端的失败重试机制，只要消息成功发送到RocketMQ，就可以认为几个分支步骤处理成功，就保证最终数据一致性

在此基础上，RocketMQ提出事务消息机制，采用两阶段提交的思路，保证主分支和consumer分支的事务一致性

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEb12d73bde8873d0d419584bd76d0435b?ynotemdtimestamp=1714350864395)

具体实现思路如下：

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE331be2d51b489b63652525dde2d43e04?ynotemdtimestamp=1714350864395)

> 1. producer发送msg到rocketmq
> 2. mq持久化消息后，向producer返回ack，把msg标记为 **暂不能投递**（半事务消息）
> 3. producer开始执行本地事务逻辑
> 4. proucer向服务端提交事务结果(二次确认，Commit或是Rollback)，服务端处理逻辑如下：
>    1. commit: 服务端将半事务消息标记为可投递，投递给consumer
>    2. rollback：服务端回滚事务，不会将半事务消息投递给consumer
> 5. 在断网或producer重启的情况下，服务端对二次确认结果为 Unknown 状态，经过固定时间后，服务端将对producer(任一实例)发起消息回查
> 6. producer收到消息回查后，需要检查对应消息的本地事务执行的最终结果
> 7. producer根据检查到的本地事务最终状态，再次提交二次确认，服务端再按步骤4进行处理

**示例代码**

```java
package org.apache.rocketmq.example.transaction;

import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.client.producer.TransactionListener;
import org.apache.rocketmq.client.producer.TransactionMQProducer;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

import java.io.UnsupportedEncodingException;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TransactionProducer {

    public static final String PRODUCER_GROUP = "please_rename_unique_group_name";
    public static final String DEFAULT_NAMESRVADDR = "127.0.0.1:9876";
    public static final String TOPIC = "TopicTest1234";

    public static final int MESSAGE_COUNT = 10;

    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer(PRODUCER_GROUP);

        // Uncomment the following line while debugging, namesrvAddr should be set to your local address
//        producer.setNamesrvAddr(DEFAULT_NAMESRVADDR);
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<>(2000), r -> {
            Thread thread = new Thread(r);
            thread.setName("client-transaction-msg-check-thread");
            return thread;
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < MESSAGE_COUNT; i++) {
            try {
                Message msg =
                    new Message(TOPIC, tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}
```

**实现重点**

使用 **TransactionMQProdcer**，注入一个 **TransactionListener** 来执行本地事务，以及后续对本地事务的检查

**注意点**

1. 半消息是对消费者不可见的一种消息，实际上，RocketMQ是将消息转发到了一个系统Topic，**RMQ_SYS_TRANS_HALF_TOPIC**
2. 事务消息中，本地事务回查次数通过参数 **transactionCheckMax** 设定，默认15次，本地事务回查的间隔通过参数 **transactionCheckInterval** 设定，默认60s。超过回查次数后，消息会被丢弃
3. 了解事务消息机制后，在具体执行时，可以对事务流程进行适当调整

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEc3473961a0c3069990bc54cee2a6f275?ynotemdtimestamp=1714350864395)

4. 通过下面的题目理解RocketMQ事务消息机制的作用

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE6687e8aeadfaf8ebd5d98b9054af5f0a?ynotemdtimestamp=1714350864395)

### 9. ACL权限控制机制

**应用场景**

RocketMQ提供了针对 队列、用户 等不同维度的权限管理机制

通常来说，RocketMQ作为一个内部服务，是不需要进行权限控制的，但通过RocketMQ进行跨部门甚至跨公司的合作，权限控制的重要性就凸显出来了

**权限控制体系**

1. RocketMQ针对每个Topic，都有完整的权限控制
   1. **perm**：表示Topic的权限，有3个可选项
      - 2：禁写禁订阅
      - 4：可订阅，不能写
      - 6：可写可订阅

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCEef9208a4e84e3d5a70c5b854d04a8da7?ynotemdtimestamp=1714350864395)

2. Broker端提供了更详细的权限控制，在 **broker.conf** 设置 **aclEnable=true**，再通过修改 **plain_acl.yml(热加载)** 进行权限配置。

```yaml
#全局白名单，不受ACL控制
#通常需要将主从架构中的所有节点加进来
globalWhiteRemoteAddresses:
- 10.10.103.*
- 192.168.0.*

accounts:
#第一个账户
- accessKey: RocketMQ
  secretKey: 12345678
  whiteRemoteAddress:
  admin: false 
  defaultTopicPerm: DENY #默认Topic访问策略是拒绝
  defaultGroupPerm: SUB #默认Group访问策略是只允许订阅
  topicPerms:
  - topicA=DENY #topicA拒绝
  - topicB=PUB|SUB #topicB允许发布和订阅消息
  - topicC=SUB #topicC只允许订阅
  groupPerms:
  # the group should convert to retry topic
  - groupA=DENY
  - groupB=PUB|SUB
  - groupC=SUB
#第二个账户，只要是来自192.168.1.*的IP，就可以访问所有资源
- accessKey: rocketmq2
  secretKey: 12345678
  whiteRemoteAddress: 192.168.1.*
  # if it is admin, it could access all resources
  admin: true
```

3. 客户端通过 **accessKey和secretKey** 提交身份信息

```xml
<dependency>
 <groupId>org.apache.rocketmq</groupId>
 <artifactId>rocketmq-acl</artifactId>
 <version>${lastVersion}</version>
</dependency>
```

4. 在申明客户端时，传入一个RPCHook

```java
//声明时传入RPCHook
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName", getAclRPCHook());

private static final String ACL_ACCESS_KEY = "RocketMQ";
private static final String ACL_SECRET_KEY = "1234567";

static RPCHook getAclRPCHook() {
  return new AclClientRPCHook(new SessionCredentials(ACL_ACCESS_KEY,ACL_SECRET_KEY));
}
```

## 2. SpringBoot整合RocketMQ

### 1. 快速实战

**依赖**

注意版本是否有冲突

```xml
<dependencies>
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.2.2</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.rocketmq</groupId>
                    <artifactId>rocketmq-client</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-client</artifactId>
            <version>4.9.5</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.5.9</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <version>2.5.9</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
    </dependencies>
```

**配置**

```properties
rocketmq.name-server=192.168.65.112:9876
rocketmq.producer.group=springBootGroup

#如果这里不配，那就需要在消费者的注解中配。
#rocketmq.consumer.topic=
rocketmq.consumer.group=testGroup
server.port=9000
```

**producer**

rocketMQTemplate不光可以发消息，还可以主动拉消息

```java
package com.roy.rocketmq.basic;

import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.apache.rocketmq.spring.support.RocketMQHeaders;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;

/**
 * @author ：楼兰
 * @description:
 **/
@Component
public class SpringProducer {

    @Resource
    private RocketMQTemplate rocketMQTemplate;

    public void sendMessage(String topic,String msg){
        this.rocketMQTemplate.convertAndSend(topic,msg);
    }
}
```

**consumer**

```java
@Component
@RocketMQMessageListener(consumerGroup = "MyConsumerGroup", topic = "TestTopic",consumeMode= ConsumeMode.CONCURRENTLY,messageModel= MessageModel.BROADCASTING)
public class SpringConsumer implements RocketMQListener<String> {
    @Override
    public void onMessage(String message) {
        System.out.println("Received message : "+ message);
    }
}
```

SpringBoot框架中对消息的封装与原生API是不一样的

### 2. 如何处理各种消息类型

1. 基础消息发送机制参考：com.roy.rocketmq.SpringRocketTest
2. 一个RocketMQTemplate实例只能包含一个producer，也就只能往一个Topic下发送消息。如果需要往另外一个Topic发消息，需要通过 **@ExtRocketMQTemplateConfiguration()注解另外声明一个子类实例**
3. 对于事务消息机制，最关键的事务监听需要通过 **@RocketMQTransactionListener** 注入到Spring容器中。在该注解通过 **rocketMQTemplateBeanName** 属性，指向具体的RocketMQTemplate子类

### 3. 实现原理

#### 1. push模式

push模式对@RocketMQMessageListener注解的处理方式，入口在rocketmq-spring-boot-2.2.2.jar中的**ListenerContainerConfiguration**类中

该类继承了Spring当中的 **SmartInitializingSingleton** 接口，当Spring容器当中所有非懒加载的实例加载完成后，就会触发 **afterSingletonsInstantiated** 方法进行初始化

该方法中，会扫描所有带有注解@RocketMQMessageListener注解的类，将他注册到内部一个Container容器中

```java
public void afterSingletonsInstantiated() {
  Map<String, Object> beans = this.applicationContext.getBeansWithAnnotation(RocketMQMessageListener.class)
    .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

  beans.forEach(this::registerContainer);
}
```

这个Container可以认为是客户端实例的一个容器，用来封装RocketMQ的原生API

```java
private void registerContainer(String beanName, Object bean) {
       .....
 //获取Bean上面的注解
        RocketMQMessageListener annotation = clazz.getAnnotation(RocketMQMessageListener.class);

     ...
    //检查注解的配置情况
        validate(annotation);

        String containerBeanName = String.format("%s_%s", DefaultRocketMQListenerContainer.class.getName(),
            counter.incrementAndGet());
        GenericApplicationContext genericApplicationContext = (GenericApplicationContext) applicationContext;
  //将扫描到的注解转化成为Container，并注册到上下文中。
        genericApplicationContext.registerBean(containerBeanName, DefaultRocketMQListenerContainer.class,
            () -> createRocketMQListenerContainer(containerBeanName, bean, annotation));
        DefaultRocketMQListenerContainer container = genericApplicationContext.getBean(containerBeanName,
            DefaultRocketMQListenerContainer.class);
     //启动容器，这里就相当于是启动了消费者
        if (!container.isRunning()) {
            try {
                container.start();
            } catch (Exception e) {
                log.error("Started container failed. {}", container, e);
                throw new RuntimeException(e);
            }
        }

        log.info("Register the listener to container, listenerBeanName:{}, containerBeanName:{}", beanName, containerBeanName);
    }
```

这里最关注的方法 **createRocketMQListenerContainer**，基本看不到RocketMQ的原生API，都是在创建和维护一个 **DefaultRocketMQListenerContainer** 对象

该对象实现了 **InitializingBean**，里面有Spring提供的对象初始化的扩展机制方法 **afterPropertiesSet**

```java
public void afterPropertiesSet() throws Exception {
  initRocketMQPushConsumer();

  this.messageType = getMessageType();
  this.methodParameter = getMethodParameter();
  log.debug("RocketMQ messageType: {}", messageType);
}
```

此方法用来初始化RocketMQ消费者

```java
private void initRocketMQPushConsumer() throws MQClientException {
       .....
        //检查并创建consumer对象。
        if (Objects.nonNull(rpcHook)) {
            consumer = new DefaultMQPushConsumer(consumerGroup, rpcHook, new AllocateMessageQueueAveragely(),
                enableMsgTrace, this.applicationContext.getEnvironment().
                resolveRequiredPlaceholders(this.rocketMQMessageListener.customizedTraceTopic()));
            consumer.setVipChannelEnabled(false);
        } else {
            log.debug("Access-key or secret-key not configure in " + this + ".");
            consumer = new DefaultMQPushConsumer(consumerGroup, enableMsgTrace,
                this.applicationContext.getEnvironment().
                    resolveRequiredPlaceholders(this.rocketMQMessageListener.customizedTraceTopic()));
        }
        // 定制instanceName，有没有很熟悉！！！
        consumer.setInstanceName(RocketMQUtil.getInstanceName(nameServer));
  .....
        //设定广播消费还是集群消费。
        switch (messageModel) {
            case BROADCASTING:
                consumer.setMessageModel(org.apache.rocketmq.common.protocol.heartbeat.MessageModel.BROADCASTING);
                break;
            case CLUSTERING:
                consumer.setMessageModel(org.apache.rocketmq.common.protocol.heartbeat.MessageModel.CLUSTERING);
                break;
            default:
                throw new IllegalArgumentException("Property 'messageModel' was wrong.");
        }
     //维护消费者的其他属性。   
     ...
           //指定Consumer的消费监听 --》在消费监听中就会去调用onMessage方法。
           switch (consumeMode) {
            case ORDERLY:
                consumer.setMessageListener(new DefaultMessageListenerOrderly());
                break;
            case CONCURRENTLY:
                consumer.setMessageListener(new DefaultMessageListenerConcurrently());
                break;
            default:
                throw new IllegalArgumentException("Property 'consumeMode' was wrong.");
        }
    }
```

#### 2. pull模式

通过RocketMQTemplate实例注入一个 **DefaultLitePullConsumer** 实例实现

注入启动该实例后，后续通过template.receive()来调用该实例的poll方法，主动去pull消息

初始化该实例代码在rocketmq-spring-boot-2.2.2.jar中的**RocketMQAutoConfiguration**类中

该配置类在jar包中的spring.factories文件中，通过SpringBoot的自动装载机制加载进来

```java
@Bean(CONSUMER_BEAN_NAME)
    @ConditionalOnMissingBean(DefaultLitePullConsumer.class)
    @ConditionalOnProperty(prefix = "rocketmq", value = {"name-server", "consumer.group", "consumer.topic"}) //解析的springboot配置属性。
    public DefaultLitePullConsumer defaultLitePullConsumer(RocketMQProperties rocketMQProperties)
            throws MQClientException {
        RocketMQProperties.Consumer consumerConfig = rocketMQProperties.getConsumer();
        String nameServer = rocketMQProperties.getNameServer();
        String groupName = consumerConfig.getGroup();
        String topicName = consumerConfig.getTopic();
        Assert.hasText(nameServer, "[rocketmq.name-server] must not be null");
        Assert.hasText(groupName, "[rocketmq.consumer.group] must not be null");
        Assert.hasText(topicName, "[rocketmq.consumer.topic] must not be null");
  
        ...
        //创建消费者   
        DefaultLitePullConsumer litePullConsumer = RocketMQUtil.createDefaultLitePullConsumer(nameServer, accessChannel,
                groupName, topicName, messageModel, selectorType, selectorExpression, ak, sk, pullBatchSize, useTLS);
        litePullConsumer.setEnableMsgTrace(consumerConfig.isEnableMsgTrace());
        litePullConsumer.setCustomizedTraceTopic(consumerConfig.getCustomizedTraceTopic());
        litePullConsumer.setNamespace(consumerConfig.getNamespace());
        return litePullConsumer;
    }
```

RocketMQUtil.createDefaultLitePullConsumer()就是在维护一个DefaultLitePullConsumer实例

该实例就是RocketMQ原生API提供的拉模式客户端

> 实际开发中，拉模式用得较少，但RocketMQ针对拉模式做了非常多的优化
>
> 原本提供了一个DefaultMQPullConsumer，进行拉模式消费
>
> DefaultLitePullConsumer在此基础上，做了很多优化

## 3. RocketMQ最佳实践

### 1. 合理分配Topic、Tag

一个app尽量用一个topic，消息子类型用tag来标识，consumer利用tag通过broker做消息过滤

> kafka一大问题就是Topic过多，造成partiton文件过多，影响性能，而RocketMQ的Topic完全不会对消息转发性能有影响。
>
> 但是Topic过多，还是会加大RocketMQ的元数据维护的性能消耗，所以仍然需要合理分配。
>
> 使用tag区分消息，尽量就用tag过滤，不要使用复杂的sql过滤，
>
> 消息过滤虽减少网络io，但会加大broker端的消息处理压力，所以，过滤逻辑越简单约好

### 2. 使用key加快消息索引

key也可以参与消息过滤，通常建议每个msg分配一个唯一key。

1. 配合tag进行更精确的消息过滤
2. broker会为每个msg建一个hash索引，可以通过topic、key来查某条历史的消息内容、处理情况，在控制台就可以看到。为了减少hash冲突，官方建议尽量保持key的唯一性

### 3. 关注错误消息重试

msg处理失败，broker会做消息重新投递。实际上，roketmq为每个consumer group组建一个对应的重试队列。

重试的消息进入 **%RETRY% + ConsumerGroup** 的队列中

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE77a358d80e635c8c27a132711a67766d?ynotemdtimestamp=1714350864395)

多关注重试队列，里面出现大量消息时，意味着consumer出现问题，要及时跟踪干预

rocketmq默认允许msg最多重试16次：

| 重试次数 | 与上次重试的间隔时间 | 重试次数 | 与上次重试的间隔时间 |
| :------: | :------------------: | :------: | :------------------: |
|    1     |        10 秒         |    9     |        7 分钟        |
|    2     |        30 秒         |    10    |        8 分钟        |
|    3     |        1 分钟        |    11    |        9 分钟        |
|    4     |        2 分钟        |    12    |       10 分钟        |
|    5     |        3 分钟        |    13    |       20 分钟        |
|    6     |        4 分钟        |    14    |       30 分钟        |
|    7     |        5 分钟        |    15    |        1 小时        |
|    8     |        6 分钟        |    16    |        2 小时        |

> 此重试时间取的延迟消息级别后16位

重试16次仍然失败，消息转入死信队列

可通过 **consumer.setMaxReconsumeTimes(x)** 设置重试次数。超过16次后，每次间隔为2h

**配置覆盖**

msg最大重试次数的设置对相同GroupID下的所有consumer有效，并且最后启动的consumer会覆盖之前启动的consumer的配置

### 4. 手动处理死信队列

通常，msg进入死信队列意味着在consumer的过程出现了较为严重的错误，并且无法自行恢复。

此时，一般需要人工处理死信消息，比如修复consumer后，转发到正常的topic重新消费，或者丢弃。

死信队列名称: **%DLQ% + ConsumGroup**

![](https://note.youdao.com/yws/public/resource/132b7cd6003f68f88986679e8edeb3cf/WEBRESOURCE1daaf3d4b79b66ee958b31522856fc65?ynotemdtimestamp=1714350864395)

**死信队列的特征：**

- 一个死信队列包含一个ConsumGroup，而不是对应某个consumer实例
- 如果一个ConsumGroup没有产生死信队列，RocketMQ就不会为其创建相应的死信队列
- 一个死信队列包含了这个ConsumGroup里的所有死信消息，而不区分msg属于哪个topic
- 死信队列中的msg不会再被consumer正常消费
- 死信队列的有效期跟正常msg相同，默认3天。对应broker.conf中的 **fileReservedTime属性**。超过该时间的msg会被删除，不管msg是否消费过

### 5. 消费者端进行幂等控制

在MQ系统中，对于消息幂等有3种实现语义：

- at most once 最多一次：每条消息最多被消费一次
  - 最好保证，mq直接异步发送、sendOneWay等方式就可以保证
- at least once 至少一次：每条消息至少被消费一次
  - 同步发送、事务消息等方式保证
- exactly once 刚好一次：每条消息都只会确定的消费一次
  - 最难保证，mq只能保证 at least once，需要业务自己保证幂等
  - 阿里云商业版有明确API支持。

**消息幂等的必要性**

mq消息可能重复，简单概括以下几种情况：

- 发送时消息重复
  - msg -> broker，broker持久化，响应成功前网络闪断或客户端宕机，导致服务端响应失败。
  - producer意识到消息失败并重推，服务端就会收到两条一样的消息，consumer也是
- 投递时消息重复
  - broker -> consumer，consumer完成业务处理，响应成功前出问题导致失败
  - broker为保证msg至少被消费一次，重推msg，consumer就会收到两条一样的消息
- 负载均衡时消息重复（包括但不限于网络抖动、Broker重启以及订阅方应用重启）
  - broker或客户端重启、扩容、缩容时，会触发 rebalance，此时consumer可能收到重复msg

**处理方式**

业务保证，每条msg的messageId保证唯一，就算多次投递，该messageId也一致

业务上用此 messageId 来判断是否处理成功过