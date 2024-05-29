[TOC]

# 楼兰 - RocketMQ快速实战与集群架构原理详解

## 1. MQ简介

消息队列：MessageQueue

主要作用：

**异步、解耦、削峰**

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCEaeedf0a36b4757196364ba5882e0addf?ynotemdtimestamp=1713352987015)

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE76c2e512b8a6344c5f9a09b444c75ef8?ynotemdtimestamp=1713352987015)

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCEb3d6cdd33d64b33ec8c923e3e6350418?ynotemdtimestamp=1713352987015)

## 2. RocketMQ产品特点

阿里产品，经历阶段：

1. 早期用ActiveMQ，达到性能瓶颈，换产品
2. 用Kafka，Topic过多时，Partition文件过多，加大文件索引的耗时，严重影响IO性能
3. 自研，早期叫MetaQ，后来改名叫RocketMQ

最早预期：解决多Topic下的IO性能压力。后期不断改进，成了Apache的顶级项目

| 优点            | 缺点                                                         | 适合场景                                 |                                  |
| --------------- | ------------------------------------------------------------ | ---------------------------------------- | -------------------------------- |
| Apache Kafka    | 吞吐量非常大，性能非常好，集群高可用。                       | 会有丢数据的可能，功能比较单一           | 日志分析、大数据采集             |
| RabbitMQ        | 消息可靠性高，功能全面。                                     | erlang语言不好定制。吞吐量比较低。       | 企业内部小规模服务调用           |
| Apache Pulsar   | 基于Bookeeper构建，消息可靠性非常高。                        | 周边生态还有差距，目前使用的公司比较少。 | 企业内部大规模服务调用           |
| Apache RocketMQ | 高吞吐、高性能、高可用。功能全面。客户端协议丰富。使用java语言开发，方便定制。 | 服务加载比较慢。                         | **几乎全场景，特别适合金融场景** |

## 3. RocketMQ快速实战

### 1. 资料&版本

[官网](http://rocketmq.apache.org)

[下载](https://rocketmq.apache.org/download/)

最新版本5.x，此课讲基于4.9.5（企业版本升级有滞后性）

5.x 新特性简介：

1. **针对定时消息**，4.x只能设定几个固定的延迟级别，5.x可以指定具体发送时间
2. **针对客户端语言**，4.x只支持基于Netty的Java客户端，5.x支持Grpc
3. **针对服务架构**，4.x只支持固定角色的普通集群和可以动态切换角色的Dledger集群，5.x支持Dledger Controlelr混合集群模式，即可以**混合使用Dledger的集群机制以及Broker本地的文件管理机制**

### 2. 下载运行

![1713404936159.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713404936159.png)

包介绍：

1. benchmark：压测脚本
2. bin：执行脚本
3. conf：配置文件
4. lib：运行jar包

RocketMQ默认启动内存需要12G，通过修改参数设置（本机实操用，生产看资源额度）

```shell
vim rocketmq-all-4.9.5-bin-release/bin/runserver.sh
```

![1713405128149.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713405128149.png)

```shell
vim rocketmq-all-4.9.5-bin-release/bin/runbroker.sh
```

![1713405235831.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713405235831.png)

启动**nameserver**服务（以下都需要JDK环境）

```apl
nohup rocketmq-all-4.9.5-bin-release/bin/mqnamesrv &
```

![1713405554461.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713405554461.png)

启动**broker**服务

先做个配置(允许broker端自动创建新的Topic)

**如果服务器有多张网卡(云服务器一般都有外网网卡和内网网卡)，需要再增加 brokerIP1 属性，指向目标网卡，不然默认随机选择一张网卡(比如选了内网网卡，外网就访问不到了)**

```shell
vim rocketmq-all-4.9.5-bin-release/conf/broker.conf
```

![1713405705216.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713405705216.png)

```apl
nohup rocketmq-all-4.9.5-bin-release/bin/mqbroker &
```

![1713405885297.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713405885297.png)

注：

实际部署时，通常添加部署地址(改自己实际路径)到环境变量,后续直接使用bin/下命令即可。

**export NAMESRV_ADDR='localhost:9876'** 用于后面测试

jdk17后面用于dashboad maven编译打包不行，降版本到jdk11后可以了

```shell
vim ~/.bash_profile

export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-17.jdk/Contents/Home
export PATH="$JAVA_HOME/bin:$PATH"
export ROCKETMQ_HOME=/Users/agefades/Documents/Platform/rocketmq/rocketmq-all-4.9.5-bin-release
export PATH="$ROCKETMQ_HOME/bin:$PATH"
export NAMESRV_ADDR='localhost:9876'
```

![1713410163004.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713410163004.png)

通过mqshutdown停止服务

```apl
mqshutdown namesrv # 关闭nameserver服务

mqshutdown broker # 关闭broker服务
```

### 3. 消息收发

```apl
# bin下工具，发送1000条，速度很快
tools.sh org.apache.rocketmq.example.quickstart.Producer 
```

![1713410358399.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713410358399.png)

```apl
# 消费,指令不会直接结束,而是继续等待消费,直接control+c结束即可
tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

![1713410473039.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713410473039.png)

### 4. 部署Web

![1713410754103.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713410754103.png)

下载源码、Maven编译打包、运行Jar

```apl
mvn clean package -Dmaven.test.skip=true
```

![1713410855522.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713410855522.png)

在jar包所在目录改配置，启动jar

我用yml配置文件启动报错，端口改了下，默认是8080

```bash
vim application.properties
```

```properties
rocketmq.config.namesrvAddrs[0]=localhost:9876
server.port=8888
```

```shell
# 启动jar
nohup java -jar rocketmq-dashboard-1.0.0.jar &
```

![1713426364060.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1713426364060.png)

### 5. 升级集群

RocketMQ分布式集群基于主从架构，master响应客户端请求，slave备份master的数据

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE828dd3f51f63d6642877ad99c28514b9?ynotemdtimestamp=1713412580886)

集群不实操了，仅记录配置项

| 机器名  | nameServer服务部署 | broker服务部署      |
| ------- | ------------------ | ------------------- |
| worker1 | nameServer         |                     |
| worker2 | nameServer         | broker-a,broker-b-s |
| worker3 | nameServer         | broker-a-s,broker-b |

#### 1. 部署nameServer

无额外配置，如上记录部署即可

#### 2. 对Broker服务进行集群配置

在conf目录下，提供了多种集群的部署配置文件模板

##### 2m-noslave

2主无从，存在单点故障

##### 2m-2s-async和2m-2s-sync

2主2从，异步或同步

##### dledger

具备主从切换的高可用集群。集群节点基于**Raft协议**随机选举一个Leader，其余节点均为follower



这里用2m-2s-async方式，仅记录下主要配置文件

```properties
#所属集群名字，名字一样的节点就在同一个集群内
brokerClusterName=rocketmq-cluster
#broker名字，名字一样的节点就是一组主从节点。
brokerName=broker-a
#brokerid,0就表示是Master，>0的都是表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=worker1:9876;worker2:9876;worker3:9876
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
deleteWhen=04
fileReservedTime=120
#存储路径
storePathRootDir=/app/rocketmq/store
storePathCommitLog=/app/rocketmq/store/commitlog
storePathConsumeQueue=/app/rocketmq/store/consumequeue
storePathIndex=/app/rocketmq/store/index
storeCheckpoint=/app/rocketmq/store/checkpoint
abortFile=/app/rocketmq/store/abort
#Broker 的角色
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
#Broker 对外服务的监听端口
listenPort=10911
```

#### 3. 启动Broker服务

```bash
cd /app/rocketmq/rocketmq-all-4.9.5-bin-release
nohup bin/mqbroker -c ./conf/2m-2s-async/broker-a.properties &
nohup bin/mqbroker -c ./conf/2m-2s-async/broker-b-s.properties &
```

#### 4. 检查集群状态

执行mqadmin命令，需在机器上配置了NAMESRV环境变量

```bash
[oper@worker1 bin]$ cd /app/rocketmq/rocketmq-all-4.9.5-bin-release/bin
[oper@worker1 bin]$ mqadmin clusterList
RocketMQLog:WARN No appenders could be found for logger (io.netty.util.internal.InternalThreadLocalMap).
RocketMQLog:WARN Please initialize the logger system properly.
#Cluster Name     #Broker Name            #BID  #Addr                  #Version                #InTPS(LOAD)       #OutTPS(LOAD) #PCWait(ms) #Hour #SPACE
rocketmq-cluster  broker-a                0     192.168.232.129:10911  V4_9_1                   0.00(0,0ms)         0.00(0,0ms)          0 3425.28 0.3594
rocketmq-cluster  broker-a                1     192.168.232.130:11011  V4_9_1                   0.00(0,0ms)         0.00(0,0ms)          0 3425.28 0.3607
rocketmq-cluster  broker-b                0     192.168.232.130:10911  V4_9_1                   0.00(0,0ms)         0.00(0,0ms)          0 3425.27 0.3607
rocketmq-cluster  broker-b                1     192.168.232.129:11011  V4_9_1                   0.00(0,0ms)         0.00(0,0ms)
```

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE91238717cfe2d5fc2eca16b5b74d448d?ynotemdtimestamp=1713412580886)

### 6. 升级高可用集群

主从架构的RocketMQ集群，只会有slave备份数据，broker宕机后，集群会缺失这部分数据，slave并不会自动升级响应请求

**Dledger**集群是具备角色自动转换功能的高可用集群

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE42440082d829ae33eacab76a0287fa96?ynotemdtimestamp=1713859840296)

不再单独指定各个broker，而是由它们自行选举，基于Raft协议，投票数大于半数则当选（集群基数节点）

leader响应客户端，follower备份、在leader宕机时参与选举

具体配置不再记录，有兴趣的可自行翻阅官方文档

**ChatGPT对Dledger的描述**

> RocketMQ是一款分布式消息队列系统，主要用于处理大量数据的实时传输和处理。在RocketMQ中，DLedger是一个为高可用、高性能、高可靠的分布式存储系统提供基础支持的组件。DLedger集群主要具有以下功能：
>
> 1. 数据复制：DLedger集群通过raft协议来保证数据的一致性。在集群中，每个节点都维护一个相同的数据副本，以确保当某个节点出现故障时，数据不会丢失。
> 2. 容错性：DLedger集群具有很高的容错性。即使集群中的部分节点发生故障，只要集群中有大多数节点（即超过半数）仍在正常工作，整个集群将继续提供服务。
> 3. 高可用性：DLedger集群通过负载均衡和热备份等机制，确保在节点故障时能够快速切换到其他正常节点，提高整个系统的可用性。
> 4. 分布式锁：DLedger集群提供分布式锁功能，可以解决分布式系统中的资源争用问题，实现跨节点的资源同步。
> 5. 强一致性：DLedger集群通过使用Raft一致性协议，确保在多个副本节点之间同步数据，保证数据的强一致性。
> 6. 高性能：DLedger集群支持水平扩展，可以通过增加节点来提高系统的吞吐量和存储能力，以满足不断增长的业务需求。
> 7. 易于管理：DLedger集群提供了一系列管理和监控功能，使运维人员可以更方便地掌握系统的运行状况，及时发现和解决问题。
>
> 总之，RocketMQ的DLedger集群具有高可用、高性能、高可靠等特点，为分布式消息队列系统提供了坚实的基础。

> 问题：Dledger集群如何防止集群脑裂问题？
>
> DLedger集群通过使用Raft协议来防止集群脑裂（split-brain）问题。脑裂问题是指在分布式系统中，由于网络分区或其他原因导致集群被分割成两个或多个子集群，各自独立运行且无法感知到其他子集群的存在。这可能导致数据不一致和错误决策。Raft协议采用了一系列措施来避免脑裂问题的发生：
>
> 1. 选举机制：Raft协议的基础是选举出一个领导者（Leader），其他节点（Follower）都从领导者获取数据。选举过程要求候选人必须获得集群中大多数节点的支持才能成为领导者。这确保了集群中只能有一个领导者，从而避免了脑裂问题。
> 2. 任期（Term）：Raft协议为每个选举周期设置了一个递增的任期编号。任期编号用于标识当前的领导者，确保旧的领导者不会再次被选为领导者。如果一个节点发现自己的任期小于其他节点，那么它会停止当前的工作并更新自己的任期。
> 3. 心跳机制：领导者会定期向其他节点发送心跳消息，以保持与Follower节点的连接。当一个节点长时间未收到领导者的心跳时，它会认为当前领导者失效，并启动新一轮选举。这确保了当领导者出现故障时，系统能够快速地选出新的领导者。
> 4. 日志复制：领导者负责将数据更新（日志条目）复制到其他节点。Follower节点只有在收到领导者的日志条目并将其写入本地日志后，才会响应客户端的请求。这确保了在发生脑裂情况下，不会出现多个节点试图同时修改同一份数据的情况。
>
> 通过以上措施，DLedger集群利用Raft协议避免了脑裂问题的发生，保证了系统的高可用性和数据一致性。

## 4. RocketMQ运行架构

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE5dad85a3218667451333d350c23e64a5?ynotemdtimestamp=1713859840296)

## 5. 理解RocketMQ消息模型

前提: 上面用tools.sh测试发送数据,日志如下

```apl
SendResult [sendStatus=SEND_OK, msgId=7F000001426E28A418FC6545DFD803E7, offsetMsgId=C0A8E88100002A9F0000000000B4F6E5, messageQueue=MessageQueue [topic=TopicTest, brokerName=broker-a, queueId=2], queueOffset=124]
```

这是RocketMQ的Broker服务端给消息生产者的响应，代表Broker正常接收并保存消息

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE8d342e04b0d595b842cacab00d05cf23?ynotemdtimestamp=1713859840296)

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCEb1590dd4d6bed322e722f96816134a3d?ynotemdtimestamp=1713859840296)

TopicTest下，分配了8个MessageQueue，均匀的分配在集群中的两个Broker服务下

每个队列都记录了一个最小位点和最大位点，**位点代表每个队列上存储的消息的索引，也称为偏移量offset**

每个队列都记录了125条消息，即1000条测试消息均匀分配到了这些队列

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCEa41f4ef69dd509e59375713570e4a2f7?ynotemdtimestamp=1713859840296)

消费日志如下：

```apl
ConsumeMessageThread_3 Receive New Messages: [MessageExt [brokerName=broker-b, queueId=0, storeSize=194, queueOffset=95, sysFlag=0, bornTimestamp=1666252677571, bornHost=/192.168.232.128:38414, storeTimestamp=1666252678510, storeHost=/192.168.232.130:10911, msgId=C0A8E88200002A9F0000000000B4ADD2, commitLogOffset=11840978, bodyCRC=634652396, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message{topic='TopicTest', flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=125, CONSUME_START_TIME=1666257428525, UNIQ_KEY=7F000001426E28A418FC6545DDC302F9, CLUSTER=rocketmq-cluster, TAGS=TagA}, body=[72, 101, 108, 108, 111, 32, 82, 111, 99, 107, 101, 116, 77, 81, 32, 55, 54, 49], transactionId='null'}]] 
```

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE44e0701c9c8e7937eeb675c0f65c0406?ynotemdtimestamp=1714041973261)

以消费组为单位进行订阅，代理者位点和消费者位点的差值就是没有消费完的消息

![](https://note.youdao.com/yws/public/resource/7e30a7bdff0d0ce0919fb565aa52421e/WEBRESOURCE91df231d6b3fe2c09c722f2bd7371523?ynotemdtimestamp=1714041973261)

通过offset控制每个消费组的处理进度，保证每条消息在一个消费组中只能被处理一次