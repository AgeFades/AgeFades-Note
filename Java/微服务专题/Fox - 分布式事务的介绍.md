[TOC]

# Fox - 分布式事务的介绍

[参考资料](https://zhuanlan.zhihu.com/p/353781389)

## 事务简介

- 事务（Transaction）是访问并可能更新数据库中各项数据项的一个程序执行单元（unit）
- 在关系型数据库中，一个事务由一组SQL语句组成

### 事务四大属性

- 通常称为 **ACID** 特性

#### 原子性（atomicity）

- 一个事务内的操作不可分割，要么全部完成，要么全部失败

#### 一致性（consistency）

- 一个事务，必须使数据库从一个一致性状态 变到 另一个一致性状态，事务的中间态是不能被观察到的

#### 隔离性（isolation）

- 并发执行的各个事务之间，不能互相干扰

##### 四个隔离级别

- 读未提交（read uncommitted）
- 读已提交（read committed），解决脏读
- 可重复读（repeatable read），解决不可重复读
- 串行化（serializable），解决幻读

#### 持久性（durability）

- 事务一旦提交，对数据的改变就应该是永久性的

## 本地事务

- 即操作单一数据库的事务
- 本地事务的ACID特性，直接由数据库提供支持

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/7D202956F57E42779AC15605257CFB75/17951)

### 代码案例

```java
Connection conn = ... //获取数据库连接
conn.setAutoCommit(false); //开启事务
try{
   //...执行增删改查sql
   conn.commit(); //提交事务
}catch (Exception e) {
  conn.rollback();//事务回滚
}finally{
   conn.close();//关闭链接
}
```

## 分布式事务

- 即操作多个数据库的事务

### 典型场景

#### 跨库事务

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/B6B195AF6A2948C0AF98113F40B8A2B0/17949)

#### 分库分表

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/FDD4E231830849F8B2DF575C3492B119/17950)

#### 跨服务、跨库

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/2F874908A59846E49749002AD6DE7E54/17948)

## 分布式事务的解决方案

### X/Open DTP模型与XA规范

- X/Open，现在的open group，一个独立组织，主要负责制定各种行业技术标准。
- 对分布式事务处理（DTP），提供了以下参考文档
  - [DTP参考模型](https://pubs.opengroup.org/onlinepubs/9294999599/toc.pdf)
  - [DTP XA规范](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf)

#### DTP模型五大基本元素

##### 应用程序（AP）

- 用于定义事务边界（即定义事务的开始和结束），并在事务边界内对资源进行操作

##### 资源管理器（RM）

- 如数据库、文件系统等，并提供访问资源的方式

##### 事务管理器（TM）

- 负责分配事务唯一标识，监控事务的执行进度，并负责事务的提交、回滚等

##### 通信资源管理器（CRM）

- 控制一个TM域内、或者跨TM域的分布式应用之间的通信

##### 通信协议（CP）

- 提供CRM的分布式应用节点之间的底层通信服务

#### XA规范

- 在DTP本地模型实例中，只由 AP、RM集合、TM组成。

##### 模型图

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/7C91F3E6C2EC4DE3B3CA21781F689B9D/17979)

- (1) 表示 AP - RM 的交互接口
- (2) 表示 AP - TM 的交互接口
- (3) 表示 RM - TM 的交互接口

##### 作用

- XA规范定义了 RM - TM 的交互接口（XA Interface）
- XA规范还对两阶段提交协议进行了优化

### 两阶段提交协议（2PC）

- 将 **提交(commit)** 过程划分为 **2个阶段(phase)**

#### 阶段一

- **TM通知各个RM准备提交它们的事务分支**
  - RM接收通知，判断当前工作是否可以被提交
    - 可以：RM对工作内容进行持久化，再给TM肯定答复
    - 不可以：给TM否定答复，并回滚工作内容，然后舍弃该事务分支信息
- 以 MySQL 为例，第一阶段中，事务管理器向所有涉及到的数据库服务器发出 prepare(准备提交)请求，
  - -> 数据库收到请求，执行数据修改和日志记录等处理，
  - -> 处理完成后，只是把事务的状态改成 "可以提交"，
  - -> 把结果返回给事务管理器

#### 阶段二

- **TM根据阶段一各个RM prepare的结果，决定是提交还是回滚事务**
  - 全都成功：TM 通知所有RM进行提交
  - 否则：TM 通知所有RM进行回滚
- 以 MySQL  为例
  - 如果所有 prepare 成功，事务管理器会向数据库服务器发出"确认提交"请求
    - -> 数据库把事务的 "可以提交" 状态变为 "提交完成"
    - -> 返回应答
  - 否则，数据库进行回滚

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/8E01B7FD857E47698A95041C1899DDA6/17993)

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/869BDD1F053E4E1FB02C816ED4216022/17990)

- XA 是资源层面的分布式事务，强一致性，在两阶段提交的整个过程中，一直会持有资源的锁
- TCC 是业务层面的分布式事务，最终一致性，不会一直持有资源的锁

#### 2PC存在的问题

##### 同步阻塞问题

- 2PC中全局事务的ACID特性，是依赖于各个RM的
  - 在对数据并行读写很敏感的情况下，本地事务都需将隔离级别设置为 串行化
  - 可重复读 不足以保证分布式事务的一致性
  - 然而 串行化 是执行性能最低的隔离级别

##### 单点故障

- 一旦 TM 发生故障，所有 RM 都会一直阻塞
  - 尤其在第二阶段，RM仍属于锁定事务资源的状态中，而无法继续完成事务操作
  - 就算 TM 是集群，一个 down 掉能选举出另一个，但无法解决 RM 这段时间都是处于阻塞状态的问题

##### 数据不一致

- 阶段二种，当TM向RM发送 commit 请求后，发生了局部网络异常
  - 就导致只有部分 RM 收到 commit 请求，这就造成了整个分布式系统出现数据不一致的现象 

### 三阶段提交（3PC）

- 3PC 是 2PC 的改进版本，主要两个改动点
  1. 在 TM 和 RM 中都引入了 **超时机制**
  2. 在阶段一和阶段二之间，插入一个 **准备阶段**

![](https://note.youdao.com/yws/public/resource/a55c2e084365cb61bac3ed818b93c0b7/xmlnote/2CD444984F0A4F9B85A5DBC3D4BF92CD/18033)

#### CanCommit

1. TM 向 RM 发送 CanCommit(事务询问) 请求
2. RM 收到请求后，可以顺利执行事务则返回 Yes、并进入预备状态，否则返回No

#### PreCommit

- 如果所有 RM 都响应 Yes
  1. TM 向 RM 发送 PreCommit(事务预提交) 请求，并进入 Prepared 阶段
  2. RM 收到请求后，执行事务操作，并将 undo 和 redo 记录到事务日志中
  3. 成功后，返回 ACK，开始等待最终指令
- 如果有 RM 响应 No，或者等待超时
  - TM 向 RM 发送 abort(中断) 请求
  - RM 收到请求后（或等待超时），执行事务的中断

#### DoCommit

- 执行提交
  - TM 收到 RM 的 ACK，从 预提交状态 -> 提交状态，并向 RM 发送 doCommit 请求
  - RM 收到请求后，执行正式的事务提交，完成后释放所有资源
  - RM 向 TM 响应 ACK
  - TM 收到所有 RM 的 ACK 响应后，完成事务
- 中断事务
  - TM 向所有 RM 发送 abort请求
  - RM 收到请求后，利用 PreCommit 中记录的 undo日志执行事务回滚，完成后释放所有资源
  - RM 向 TM 响应 ACK
  - TM 收到所有 RM 的 ACK 响应后，执行事务的中断

#### 3PC存在的问题

- 该阶段中，如果 RM 无法及时接受到 TM 的 doCommit 或 rebort 请求，即等待超时的情况下，默认执行 commit 
  - 就可能其他 RM 执行 abort，该节点执行了 commit，造成数据不一致问题

### 2PC 和 3PC 的区别

- 相对于 2PC，3PC主要解决单点故障问题，并减少阻塞

### 小结

- 无论是 2PC 还是 3PC 都无法彻底解决分布式事务的一致性问题
