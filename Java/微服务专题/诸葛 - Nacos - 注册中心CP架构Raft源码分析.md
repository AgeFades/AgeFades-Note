# 诸葛 - Nacos - 注册中心CP架构Raft源码分析

## 内容

- CAP与BASE原理
- Nacos、Zookeeper、Eureka的CAP架构横向对比
- Raft协议动态图解
- Nacos集群CP架构基于Raft协议源码剖析
- Nacos集群CP架构的脑裂问题
- 基于云SaaS的超大规模注册中心架构设计

## 参考资料

[Raft协议在线动态演示](http://thesecretlivesofdata.com/raft/)

[CAB与BASE图](https://www.processon.com/view/link/60dd2d07f346fb04d2d1e33c)

[主流注册中心的CAP架构选择、CP架构脑裂问题](https://www.processon.com/view/link/60dd385e0e3e745b089f6cdc)

[Nacos源码集群CP架构Raft协议实现](https://www.processon.com/view/link/60dd779c637689510d67b200)

## CAP与BASE原理

### CAP

- CAP属性：
  - Consistency：一致性
    - 节点之间数据保持强一致，不一致的节点暂停提供服务
  - Availability：可用性
    - 各节点均一直提供服务，不能保证各个节点的数据强一致
  - Partition tolerance：分区容错性
    - 集群节点间网络不可互通时，整个集群仍能对外提供服务
- CAP原则：三选二
  - P是必须分布式系统必须保证的
  - 多数时候，是在 AP 和 CP 之间做权衡选择

### BASE

- BASE属性：
  - Basically Available：基本可用
  - Soft State：软状态
  - Eventual Consistency：最终一致性
- BASE原则：
  - C、A、P三个都要，但都不能100%保证

## 主流注册中心的CAP架构选择

- MySQL单机：CA
- Eureka集群：AP
- Zookeeper集群：CP
- Nacos集群：AP或CP
- Redis集群：AP

## Raft协议的介绍

[丁凯-Raft协议详解](https://zhuanlan.zhihu.com/p/27207160)

- 简单来说，是一种分布式一致性协议，用于维护多个副本的一致性

### Raft与ZAB的异同

#### 差异

- 在进行选举操作时：
  - Raft中各节点都有随机睡眠时间，谁先苏醒则开启投票选举，其他节点均相应同意与否，来产生Leader节点
  - ZAB中，各节点同时开启选举投票，并进行票数PK，来产生Leader节点

#### 相同

- 都是分布式一致性协议 Paxos 的简化，主要包含两部分：
  - Leader选举（半数以上节点投票同意）
  - 集群写入数据同步（两阶段提交，半数以上节点写入成功）

## 集群脑裂问题

[知乎-脑裂原因及解决方法](https://zhuanlan.zhihu.com/p/74849050)

- 脑体问题举例：
  - 一个集群6个节点，3个在上海，3个在北京，1个leader、5个follower
  - 此时发生分区网络故障，将上海机房和北京机房隔离
  - 此时两个分区进行Leader重新选举，一个集群就产生了2个leader
- 解决方法举例：
  - 选举leader时，必须获得大于集群总数的一半节点投票
  - 如上例子，分区后，每边3个节点，每个节点最多获得3票，3 < 6 / 2
  - 这样就避免了脑裂问题

