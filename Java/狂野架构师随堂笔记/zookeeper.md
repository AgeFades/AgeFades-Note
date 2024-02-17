Zookeeper
概念上：分布式协调服务
本质：小型的文件存储系统 + 监听通知机制
一般存状态值或者配置信息
数据发生变更了，会将变更的东西通知给监听的客户端

CP模型 最终一致（过半机制，强一致就是要所有节点全部响应） 分区容错性

CAP定律：A是高可用性

可以做 数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁、分布式队列等功能

小型文件系统：怎么存储数据？
ZK数据结构
树形目录服务，类似于标准文件系统，基于znode进行存储，多个znode形成tree
类linux /usr/local/zookeeper
用 key-value 的形式存储
create /node1 abc  绝对路径，/node1就是key，abc 就是value
get /node1 取数据

每个路径下的节点key是唯一的（绝对路径都是唯一的）
每个节点中存储了节点value和对应的状态属性（多个）stat()
比如：节点创建时间，版本信息，事务信息...

思考：
分布式锁思路
在zk中创建子节点比如 /lock
同一时刻只能有一个节点能创建成功（绝对路径唯一），创建成功即获取锁
其他节点创建失败，就监听 /lock 节点的销毁(拿锁做完业务操作之后释放锁)

问题：
拿锁节点异常退出？不释放锁，死锁问题，其他进程还都在等待
解决：
拿锁节点异常退出也要释放锁. 抢锁建立临时节点

zk节点类型： 
持久节点 默认创建就是
持久顺序节点 create -s 在名称上加自增数字 /node1_00001 /node1_00002
临时节点 create -e 不可拥有子节点，客户端会话断开后自动删除
临时顺序节点 create -e -s 
临时节点是存在内存中的
容器节点 create -c 子节点都被删除后，容器节点也随即删除
TTL节点，客户端断开连接后不会自动删除，如果它没有子节点，并且在给定的TTL时间内无修改，就会被删除，单位是毫秒，create -t 

zk命令
help 
ls create delete deleteall递归删除 get set ..
get -w 进行监听 -s 获取状态 NodeDataChanged 
zk监听是一次性的，持续监听需要进行反复注册监听

zk java 客户端 
zk原生API session超时重连不支持、watch监听需要进行返回注册、不支持对节点的递归操作..
curator 首选客户端工具 提供各个场景对应的实现，还提供了一套 fluent 风格的API供使用
zkClient 几乎没有参考文档，异常处理简化，没有提供各个场景对应的实现
都是在zk原生API上进行更一步的封装

高级应用原理
分布式锁 数据一致性比redis强，并发不如redis
抢茅台案例：模拟下单超时问题 做秒杀 库存放在redis
不加锁：并发进来几百个线程进来拿到的都是有库存的值，比如说100，但是有500个并发线程减库存，这就是超卖 查库存和减库存操作不具备原子性

用synchronized加锁，锁的是当前整个方法，做到的是单个进程的操作原子性，分布式下一个服务多个实例，就锁不住了

zk做分布式锁，注册临时节点，注册成功即持锁，注册失败监听该节点的删除事件，节点销毁后，监听的客户端重新抢锁过程

问题：羊群效应
要是有1w个客户端线程在这里等，监听一失效通知给这1w个客户端，造成了大量的网络请求开销，也给zk很大压力

创建zk znode临时顺序节点，也是单线程做写操作的

解决：临时顺序节点
抢锁全建临时顺序节点，并把基本信息写入临时节点
客户端下一步都获取所有临时节点，判断自己是不是最小的节点，最小的即持锁
未持锁的则对前一个节点做删除时间的监听
后续就是顺序持锁

链路一个中间的节点挂了呢？删除事件也会被后一个节点监听到，也会比较所有节点，判断是否为最小，不是则继续监听前一个节点

细分：共享锁、排它锁
对读锁而言，关心前一个写锁的释放。一个写锁的释放，多个后续的读锁节点可以并发读数据
写锁，关心前一个临近节点的释放，不论读写锁，保证有序性，必须等前面的读写操作完成才能上写锁
读读能共存，读写和写写都不能共存

通过 k v 随便设个标志 r w 即区分读锁写锁

分布式锁的实现：Curator实现
InterProcessMutex 分布式可重入排它锁（可重入借助LocalMap存计数器）
InterProcessSemaphoreMutex 分布式排他锁
InterProcessMultiLock 将多个锁作为单个实体管理的容器
InterProcessReadWriteLock 分布式读写锁

可重入：比如进锁方法递归调用本身，没问题的就是可重入锁

ZK的相关高级特性：
配置管理
服务注册、服务发现
consume provider
服务启动向zk注册信息、做些心跳检测机制..

zk选举策略、集群、zab协议..
多个机器节点怎么保证数据一致
分布式一致性问题最有效的算法 Paxos，常用来做分布式选举算法
定义三种角色
发起者
投票者
不具有投票权利者
提案遵循少数服从多数、过半原则

Paxos很繁琐，推理过程很难
一般应用是对 Paxos 的简化算法，比如 zab / raft

zab：借鉴Paxos算法，专为zk设计的支持 崩溃恢复 的 原子广播 协议
三个角色
leader 维护可用的 follower 列表，负责读写操作 同步数据
follower 有投票权 能处理读操作 会把写操作转发给 leader
observer 不具有投票权 其他跟 follower 一样，提高吞吐量

client默认不会直连leader

崩溃恢复：集群中没有leader角色，就会进入崩溃恢复模式
   集群刚启动、leader失去了与过半follower的连接
   集群进入恢复，基于选举算法选举出一个新的leader，与过半follower完成数据同步，之后进入消息广播模式

原子广播：
   这个阶段，zk集群才能对外提供事务服务，并leader进行消息广播，
   有节点加入，还需要对新节点进行同步
   zab不像 2pc 需要全部 follower 都 ack，只需要过半节点的 ack 即可

选举时能否设置性能好的服务器做leader？ 可以，有选举配置项，选举对比参数，可以把性能好的服务器myid设置大一点

支持的领导选举算法
可通过ElectionAlg配置项设置zk用于选举的算法
到3.4.10版本为止：
基于udp的leaderElection
基于udp的fastLeadElection
基于udp和认证的fastLeaderelection 
基于tcp的FastLeaderElection 默认，其他三种已弃用

FastLeaderElection 原理
1. myid
每个zk数据文件夹下都需要创建一个 myid 的文件，里面写整个zk集群唯一的id(整数)
hostname 必须与 myid 一一对应
.1 .2 .3 是myid  /  zoo1 zoo2 zoo3 是hostname
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888

2. zxid
类rdbms中的事务id，标识一次更新操作的proposal id
为保证顺序，zxid必须单调递增
zk用一个64的数表示，高32位是 leader 的 epoch
从1开始，每次选出新leader，epoch加1
低32位为该epoch内的序号，每次epoch变化，都将低32位的序号重置，保证zkid的全局递增

反应出当前zk节点的数据状态，zxid值越大，代表数据越新

3. 服务器状态

FastLeaderElection 快速选举原理
zk选举会选数据比较新的节点做leader

第一轮 每个节点 投票给自己 -> 广播给其他节点
1. 选 epoch 最大的（纪元）
2. 1相等，选 zxid 最大的（最新事务）
3. 1、2都相等，选server_id最大的（zoo.cfg中的myid）

读请求：
所有节点都可正常响应

写请求：
follower -> lead -> 同步给所有follower -> follower写完就ack -> 过半ack后leader进行commit -> 广播给所有follower，响应客户端 -> follower 也进行真正的commit

跟2pc差不多，只是2pc要全部。zab只要过半