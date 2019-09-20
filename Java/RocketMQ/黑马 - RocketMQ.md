# 黑马 - RocketMQ

## 官网

[http://rocketmq.apache.org/]()

## 核心概念说明

![UTOOLS1568860537786.png](https://i.loli.net/2019/09/19/zTUdxKPnJCjHVWe.png)

```shell
# Producer
	# 消息生产者，负责产生消息，一般由业务系统负责产生消息
	# Producer Group
		# 一类 Producer 的集合名称，这类 Producer 通常发送一类消息，且发送逻辑一致。
		
# Consumer
	# 消费者，负责消费消息，一般是后台系统负责异步消费
	# Push Consumer
		# 服务端向消费端推送消息
	# Pull Consumer
		# 消费端向服务器定时拉取消息
	# Consumer Group
		# 一类 Consumer 的集合名称，通常消费一类消息，且消费逻辑一致。
	
# NameServer
	# 集群架构中的组织协调员
	# 负责 broker 的工作情况
	# 不负责消息的处理
	
# Broker
	# 是 RocketMQ 的核心负责消息的发送、接收、高可用等<真正干活的>
	# 需要定时发送自身情况到 NameServer，默认 10s 发送一次，超时2分钟会被认为该 Broker 失效
	
# Topic
	# 不同类型的消息以不同的 Topic 名称进行区分，如 User、Order 等
	# 是逻辑概念
	# Message Queue
		# 消息队列，用于存储消息
```

## Docker 安装

```shell
# 拉取镜像
docker pull foxiswho/rocketmq:server-4.3.2
docker pull foxiswho/rocketmq:broker-4.3.2

# 创建 nameServer 容器
docker run -p 9876:9876 --name rmqserver \
-e "JAVA_OPT_EXT=-server -Xms128m -Xmx128m -Xmn128m" \
-e "JAVA_OPTS=-Duser.home=/opt" \
-v /docker/rmq/rmqserver/logs:/opt/logs \
-v /docker/rmq/rmqserver/store:/opt/store \
foxiswho/rocketmq:server-4.3.2

# 创建 RocketMq-Console
docker pull styletang/rocketmq-console-ng:1.0.0
docker run -e "JAVA_OPTS=-Drocketmq.namesrv.addr=['你自己的Ip']:9876 -
Dcom.rocketmq.sendMessageWithVIPChannel=false" -p 8082:8080 -t styletang/rocketmqconsole-ng:1.0.0

# 浏览器访问Ip + Port即可
```

## Java 整合 RocketMQ

### Message 数据结构

| 字段名         | 默认值 | 说明                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| Topic          | null   | 必填，线下环境不需要申请<br />线上环境需要申请后才能使用     |
| Body           | null   | 必填，二进制形式，序列化由应用决定<br />Producer 与 Consumer 要协商好序列化形式 |
| Tags           | null   | 选填，方便服务器过滤使用 <br />目前只支持每个消息设置一个tag |
| Keys           | null   | 选填，该消息的业务关键字<br />服务器根据Keys 创建哈希索引，尽可能唯一 |
| Flag           | 0      | 选填，完全由应用来设置，RocketMQ 不做干预                    |
| DelayTimeLevel | 0      | 选填，消息延时级别，0表示不延时<br />大于0 会延时特定的时间才会被消费 |
| WaitStoreMsgOK | TRUE   | 选填，表示消息是否在服务器落盘后才返回应答                   |

```shell
# 因为 RocketMQ 是阿里系，并没有完全开源，且整合 SpringBoot 使用比较麻烦，故不再记录。
```

