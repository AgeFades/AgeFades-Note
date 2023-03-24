[TOC]

# Fox - Seata使用及原理剖析

## 资料

[官网](https://seata.io/zh-cn/index.html)

[Github 项目源码](https://github.com/seata/seata)

[官方示例Demo](https://github.com/seata/seata-samples)

## 简介

- Seata：一款开源的 **分布式事务解决方案**
  - 提供了 **AT、TCC、SAGA、XA** 等事务模式
  - **AT** 是阿里首推模式
  - 阿里云有商业GTS（Global Transaction Service : 全局事务服务）
- 版本：v1.4.0

### 三大角色

- Seata的架构中，共有三个角色如下：

#### TC

- Transaction Coordinator：事务协调者
  - 维护全局和分支事务的状态
  - 驱动全局事务提交或回滚
- 单独部署的 Server 服务端

#### TM

- Transaction Manager：事务管理器
  - 定义全局事务的范围
  - 开始、提交、回滚 全局事务
- 嵌入应用的 Client 客户端

#### RM

- Resource Manager：资源管理器
  - 管理分支事务处理的资源
  - 与 TC 交谈以 **注册分支事务** 和 **报告分支事务的状态**
  - 并驱动 **分支事务提交或回滚**
- 嵌入应用的 Client 客户端

### 一个分布式事务的生命周期

1. TM 请求 TC 开启一个全局事务
   1. -> TC  生成一个 XID 作为该全局事务的编号
      1. XID：在微服务的调用链路中传播，保证将各个子事务关联在一起
2. RM 请求 TC **将本地事务注册为全局事务的分支事务**
   1. -> 通过全局事务的 XID 进行关联
3. TM 请求 TC 告诉 XID 对应的全局事务，是进行提交还是回滚
4. TC 驱动 RM们 将 XID 对应的本地事务，是进行提交还是回滚

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/52610891D6034F9A9D4A5708BB7BF392/17464)

### 设计思路

- AT模式的核心是对业务代码无侵入，是一种改进后的两阶段提交
  - 设计思路如下：

#### 第一阶段

- 业务数据和回滚日志、记录在同一个本地事务中提交，释放本地锁、连接资源
  - 核心在于对 业务SQL 进行解析，转换成 undolog，并同时入库
  - 使用 DataSourceProxy 代理数据源完成

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/FA80EF05039A440C8D39289A88C9D528/17467)

#### 第二阶段

- 分布式事务操作成功，则 TC 通知 RM 异步删除 undolog
- 操作失败，TM 向 TC 发送回滚请求
  - -> TC 给 RM们 发回滚请求
  - -> RM 通过 XID 和 Branch ID 找到 undolog
  - -> 通过 undolog 反向生成 SQL 并执行，最终完成分支的回滚

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/428516FCC9BC444FAB52C0462289B12B/17468)

#### 整体执行流程

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/2114700EF96F426883DD0A8E9A6A24E7/17466)

### 优缺点

#### 优点

- 应用层基于 SQL解析 实现了自动补偿，最大程度降低对 业务代码 的侵入性
- TC 服务端独立部署，负责事务的注册、回滚
- 通过全局锁实现了 写隔离、读隔离

#### 缺点

- 性能损耗
  - 一条 update sql，需要做：
    1. 与 TC 通讯，获取全局事务 xid
    2. before image（解析SQL、查询一次数据库）
    3. after image（查询一次数据库）
    4. insert undo log（写一次数据库）
    5. before commit（与 TC 通讯，判断锁冲突）
  - 以上操作都需要远程通信，还是同步操作
  - 另外：undo log 写入时，blob 字段的插入性能也比较低
  - 每条写SQL都会增加这么多开销，粗略估计增加 5倍响应时间
- 性价比
  - 如上所述，引入 Seata，所有写SQL开销变大这么多倍
  - 根据二八原则，粗定 80% 操作都是没问题的，20% 操作可能有问题
  - 为了这 20% 的操作数据一致性，让 100% 写SQL 性能增加5倍左右
  - 是不是可以不引入 seata，而给这 20% 单独设计一套事后补偿机制呢？
- 全局锁
  - 相比 XA，Seata AT 虽然在一阶段成功后、就会释放数据库锁
  - 但一阶段在 commit 前、全局锁的判定也拉长了 对数据锁 的占有时间
  - 全局锁的引入实现了隔离性，但带来的问题就是阻塞、降低并发
    - 尤其是热点数据
  - Seata 在回滚时，需要先删除掉各节点的 undo log
    - 然后才能释放 TC 内存中的锁
    - 所以，第二阶段如果是回滚，锁释放的时间会更长
- 死锁问题
  - 全局锁还会增加 死锁 的风险
    - 如果出现 死锁，会不断进行重试，最终靠等待 全局锁超时 来退出
    - 这也延长了对 数据库锁 的占有时间

## 实操

### Seata Server(TC) 搭建

- 就参考官方部署文档即可，下面就不再冗余记录了

[官方部署文档](https://seata.io/zh-cn/docs/ops/deploy-guide-beginner.html)

#### Server端存储模式

- store.mode：支持三种
  - file：单机模式
    - 全局事务会话信息、内存中读写并持久化 本地文件 root.data，性能较高
  - db：高可用模式
    - 全局事务会话信息、通过 db 共享，性能相对较差
  - redis：
    - Seata Server v1.3及以上版本支持，性能较高
    - 存在事务信息丢失风险，需提前做好持久化配置

#### 资源目录

- client：
  - 存放client端sql脚本，参数配置
- config-center：
  - 各个配置中心参数导入脚本
  - config.txt(包含server和client，原名nacos-config.txt)
    - 为通用参数文件
- server：
  - 存放server端sql脚本、各个容器配置

#### DB存储模式+Nacos部署

##### 1. 下载安装包

[下载链接](https://github.com/seata/seata/releases)

- 下最新版本


##### 2. 建表(仅db模式)

- 全局事务会话信息由三项内容组成：
  - 全局事务：对应表 global_table
  - 分支事务：对应表 branch_table
  - 全局锁：对应表 lock_table
- 创建数据库 seata，执行 sql 脚本
  - srcipt/server/db/mysql.sql（seata源码sql脚本地址）
  - [1.6.1版本sql地址](https://github.com/seata/seata/tree/v1.6.1/script/server/db)

##### 3. 配置Nacos注册中心

![](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/405E40DAFC7248FA9F88987A11CFE2ED/54137)

- 配置将 Seata Server注册到Nacos，修改conf/application.yml(总的如下)

##### 4. 配置Nacos配置中心

- 修改conf/application.yml

```yaml
server:
  port: 7091

spring:
  application:
    name: seata-server

logging:
  config: classpath:logback-spring.xml
  file:
    path: ${user.home}/logs/seata
  extend:
    logstash-appender:
      destination: 127.0.0.1:4560
    kafka-appender:
      bootstrap-servers: 127.0.0.1:9092
      topic: logback_to_logstash

console:
  user:
    username: seata
    password: seata

seata:
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      # seata配置项在nacos中的命名空间
      namespace: de0bcc5c-216f-4ba1-ac55-9de09e91ce0b
      group: SEATA_GROUP
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      cluster: default
  store:
    # support: file 、 db 、 redis
    mode: file
#  server:
#    service-port: 8091 #If not configured, the default is '${server.port} + 1000'
  security:
    secretKey: SeataSecretKey0c382ef121d778043159209298fd40bf3850a017
    tokenValidityInMilliseconds: 1800000
    ignore:
      urls: /,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/api/v1/auth/login
```

##### 5. 修改存储模式、数据库连接

a) 修改/seata/script/config-center/config.txt，修改为db存储模式，并修改mysql连接配置

```properties
transport.type=TCP
transport.server=NIO
transport.heartbeat=true
transport.enableTmClientBatchSendRequest=false
transport.enableRmClientBatchSendRequest=true
transport.enableTcServerBatchSendResponse=false
transport.rpcRmRequestTimeout=30000
transport.rpcTmRequestTimeout=30000
transport.rpcTcRequestTimeout=30000
transport.threadFactory.bossThreadPrefix=NettyBoss
transport.threadFactory.workerThreadPrefix=NettyServerNIOWorker
transport.threadFactory.serverExecutorThreadPrefix=NettyServerBizHandler
transport.threadFactory.shareBossWorker=false
transport.threadFactory.clientSelectorThreadPrefix=NettyClientSelector
transport.threadFactory.clientSelectorThreadSize=1
transport.threadFactory.clientWorkerThreadPrefix=NettyClientWorkerThread
transport.threadFactory.bossThreadSize=1
transport.threadFactory.workerThreadSize=default
transport.shutdown.wait=3
transport.serialization=seata
transport.compressor=none

# 事务组
service.vgroupMapping.default_tx_group=default
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false

client.rm.asyncCommitBufferLimit=10000
client.rm.lock.retryInterval=10
client.rm.lock.retryTimes=30
client.rm.lock.retryPolicyBranchRollbackOnConflict=true
client.rm.reportRetryCount=5
client.rm.tableMetaCheckEnable=true
client.rm.tableMetaCheckerInterval=60000
client.rm.sqlParserType=druid
client.rm.reportSuccessEnable=false
client.rm.sagaBranchRegisterEnable=false
client.rm.sagaJsonParser=fastjson
client.rm.tccActionInterceptorOrder=-2147482648
client.tm.commitRetryCount=5
client.tm.rollbackRetryCount=5
client.tm.defaultGlobalTransactionTimeout=60000
client.tm.degradeCheck=false
client.tm.degradeCheckAllowTimes=10
client.tm.degradeCheckPeriod=2000
client.tm.interceptorOrder=-2147482648
client.undo.dataValidation=true
client.undo.logSerialization=jackson
client.undo.onlyCareUpdateColumns=true
server.undo.logSaveDays=7
server.undo.logDeletePeriod=86400000
client.undo.logTable=undo_log
client.undo.compress.enable=true
client.undo.compress.type=zip
client.undo.compress.threshold=64k
tcc.fence.logTableName=tcc_fence_log
tcc.fence.cleanPeriod=1h

log.exceptionRate=100

store.mode=db
store.lock.mode=file
store.session.mode=file

store.file.dir=file_store/data
store.file.maxBranchSessionSize=16384
store.file.maxGlobalSessionSize=512
store.file.fileWriteBufferCacheSize=16384
store.file.flushDiskMode=async
store.file.sessionReloadReadSize=100

store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
# 数据库配置
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true&rewriteBatchedStatements=true
store.db.user=root
store.db.password=root
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.distributedLockTable=distributed_lock
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000

store.redis.mode=single
store.redis.single.host=127.0.0.1
store.redis.single.port=6379
store.redis.maxConn=10
store.redis.minConn=1
store.redis.maxTotal=100
store.redis.database=0
store.redis.queryLimit=100

server.recovery.committingRetryPeriod=1000
server.recovery.asynCommittingRetryPeriod=1000
server.recovery.rollbackingRetryPeriod=1000
server.recovery.timeoutRetryPeriod=1000
server.maxCommitRetryTimeout=-1
server.maxRollbackRetryTimeout=-1
server.rollbackRetryTimeoutUnlockEnable=false
server.distributedLockExpireTime=10000
server.xaerNotaRetryTimeout=60000
server.session.branchAsyncQueueSize=5000
server.session.enableBranchAsyncRemove=false

metrics.enabled=false
metrics.registryType=compact
metrics.exporterList=prometheus
metrics.exporterPrometheusPort=9898
```

在store.mode=db，由于seata是通过jdbc的executeBatch来批量插入全局锁的，根据MySQL官网的说明，连接参数中的rewriteBatchedStatements为true时，在执行executeBatch，并且操作类型为insert时，jdbc驱动会把对应的SQL优化成`insert into () values (), ()`的形式来提升批量插入的性能。 根据实际的测试，该参数设置为true后，对应的批量插入性能为原来的10倍多，因此在数据源为MySQL时，建议把该参数设置为true。

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/79FDB1A3293B4913A2BFD941A3964041/53887)

b) 配置事务分组， 要与client配置的事务分组一致

- 事务分组：seata的资源逻辑，可以按微服务的需要，在应用程序（客户端）对自行定义事务分组，每组取一个名字。
- 集群：seata-server服务端一个或多个节点组成的集群cluster。 应用程序（客户端）使用时需要指定事务逻辑分组与Seata服务端集群的映射关系。

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/078050B4F6C342388EA2F0E5CD1EDA3D/53977)

事务分组如何找到后端Seata集群（TC）？

1. 首先应用程序（客户端）中配置了事务分组（GlobalTransactionScanner 构造方法的txServiceGroup参数）。若应用程序是SpringBoot则通过seata.tx-service-group 配置。
2. 应用程序（客户端）会通过用户配置的配置中心去寻找service.vgroupMapping .[事务分组配置项]，取得配置项的值就是TC集群的名称。若应用程序是SpringBoot则通过seata.service.vgroup-mapping.事务分组名=集群名称 配置
3. 拿到集群名称程序通过一定的前后缀+集群名称去构造服务名，各配置中心的服务名实现不同（前提是Seata-Server已经完成服务注册，且Seata-Server向注册中心报告cluster名与应用程序（客户端）配置的集群名称一致）
4. 拿到服务名去相应的注册中心去拉取相应服务名的服务列表，获得后端真实的TC服务列表（即Seata-Server集群节点列表）

c) 在nacos配置中心中新建配置，dataId为seataServer.properties，配置内容为上面修改后的config.txt中的配置信息

从v1.4.2版本开始，seata已支持从一个Nacos dataId中获取所有配置信息,你只需要额外添加一个dataId配置项。

- 这里用v1.6.1试了好像没用，不知道是哪里的问题，用单独配置项就可以

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/9C76805807E34504BAFE3F6A2235F15A/54003)

添加后查看：

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/870E512146BA4EF2B14F02B5610AA2E7/54005)

- 本人采取方式

```sh
# 在 seata 的这个目录下执行
seata/script/config-center/nacos
```

```shell
# 注意修改自己的 group、namespace、username、password
sh nacos-config.sh -h localhost -p 8848 -g SEATA_GROUP -t de0bcc5c-216f-4ba1-ac55-9de09e91ce0b -u nacos -w nacos
```

##### 6. 启动Seata Server

启动命令:

```shell
bin/seata-server.sh    
```

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/3F66707112A3482088AE3909878826CF/54044)

启动成功，查看控制台，账号密码都是seata。http://localhost:7091/#/login

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/55E9E1A98FDB4A2CAAF3134DC80A4935/54055)

在Nacos注册中心中可以查看到seata-server注册成功

​    ![0](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/818209B0E1D04249BA6185931497E8C4/54053)

支持的启动参数

| 参数 | 全写         | 作用                       | 备注                                                         |
| ---- | ------------ | -------------------------- | ------------------------------------------------------------ |
| -h   | --host       | 指定在注册中心注册的 IP    | 不指定时获取当前的 IP，外部访问部署在云环境和容器中的 server 建议指定 |
| -p   | --port       | 指定 server 启动的端口     | 默认为 8091                                                  |
| -m   | --storeMode  | 事务日志存储方式           | 支持file,db,redis，默认为 file 注:redis需seata-server 1.3版本及以上 |
| -n   | --serverNode | 用于指定seata-server节点ID | 如 1,2,3..., 默认为 1                                        |
| -e   | --seataEnv   | 指定 seata-server 运行环境 | 如 dev, test 等, 服务启动时会使用 registry-dev.conf 这样的配置 |

比如：

```shell
bin/seata-server.sh -p 8091 -h 127.0.0.1 -m db         
```

### SpringCloud 整合 Seata AT模式实战

![](https://note.youdao.com/yws/public/resource/c480b9d259db401acff9fdd30a770d64/xmlnote/174DF4D5850D46D4A11920DF78129626/54076)

#### 1. pom.xml

- spring-cloud-starter-alibaba-seata内部集成了seata，并实现了xid传递

```xml
<!-- seata-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>
```

#### 2. 客户端对应数据库添加undo_log表（仅AT模式）

[undo_log.sql](https://github.com/seata/seata/blob/v1.6.1/script/client/at/db/mysql.sql)

#### 3.  微服务application.yml中添加seata配置

```yaml
seata:
  application-id: ${spring.application.name}
  # seata 服务分组，要与服务端配置service.vgroup_mapping的后缀对应
  tx-service-group: default_tx_group
  config:
    # support: nacos, consul, apollo, zk, etcd3
    type: nacos
    nacos:
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      namespace: de0bcc5c-216f-4ba1-ac55-9de09e91ce0b
  registry:
    # support: nacos, eureka, redis, zk, consul, etcd3, sofa
    type: nacos
    nacos:
      application: seata-server
      server-addr: 127.0.0.1:8848
      group: SEATA_GROUP
      cluster: default
```

#### 4. 在全局事务发起者中添加@GlobalTransactional注解

- 写的测试代码

```java
@Override
@Transactional
@GlobalTransactional(name = "createOrder", rollbackFor = Exception.class)
public void add(SysUserAddReq req) {
  // 1. 唯一索引校验
  Assert.isTrue(lambdaQuery().eq(SysUser::getUsername, req.getUsername()).count() == 0, "用户名重复");

  // 2. 密码加密
  req.setPassword(DigestUtil.bcrypt(req.getPassword()));

  // 3. 保存用户
  SysUser sysUser = BeanUtil.copyProperties(req, SysUser.class);
  String id = IdUtil.getSnowflake().nextIdStr();
  log.info("测试自己赋值主键: {}", id);
  sysUser.setId(id);
  save(sysUser);

  // 这里feign发起远程调用测试保存
  testClient.saveOrder();

  // 手动创建异常情况
  int i = 1 / 0;
}
```

#### 5. 测试结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1678265841323.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1678265870953.png)

- 两边服务都做了回滚操作，分布式事务成功
- 把 @GlobalTransactional 注释测试，订单数据不会回滚，用户数据回滚了

#### 6. 问题记录

- 调用加了全局事务注解的方法，报错如下：
  - Seata 客户端服务端配置不正确，具体哪里不准确得自己慢慢排查，反正就是配的不对就报这些错，网上一搜什么资料就是看源码自己调试 巴拉巴拉的，Seata本身配置就繁琐，各个版本细节差异还多，报个异常信息也不友好，不能一下get到问题点

```shell
io.seata.common.exception.FrameworkException: No available service
```

- 客户端服务持续间隔打印error日志
  - 同上，一样是配置不正确

```shell
no available service 'null' found, please make sure registry config correct
```

## Seata XA模式

### 整体机制

- 在 Seata 定义的分布式事务框架内，利用事务资源（数据库、服务消息等）对 XA 协议的支持，以 XA 协议的机制来管理分支事务的一种事务模式

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/E2227F72EB4741EFB2044CFDA2BC6811/55865)

- AT 和 XA 模式数据源代理机制对比

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/FD20DEB1A7494ED5A792985AB1933CA9/55847)

- XA 模式的使用
  - 从编程模型上，XA模式与AT模式保持完全一致
  - 只需要修改数据源代理，即可实现 XA 模式与 AT 模式之间的切换

```java
@Bean("dataSource")
public DataSource dataSource(DruidDataSource druidDataSource) {
    // DataSourceProxy for AT mode
    // return new DataSourceProxy(druidDataSource);

    // DataSourceProxyXA for XA mode
    return new DataSourceProxyXA(druidDataSource);
}
```

### Spring Cloud整合Seata XA实战

- 对比 Seata AT 模式配置，只需要修改两个地方：

  - 数据库中不需要 undo_log 表，undo_log 表仅用于 AT 模式

  - **修改数据源代码模式为 XA 模式**

    - ```yaml
      seata:
        # 数据源代理模式 默认AT
        data-source-proxy-mode: XA
      ```

## 什么是 TCC

- TCC 是基于分布式事务中的 **二阶段提交协议** 实现
- Try：
  - 对业务资源的检查并预留
- Confirm：
  - 对业务处理进行提交，即 commit 操作，只要 Try 成功，那么该步骤一定成功
- Cancel：
  - 对业务处理进行取消，即 rollback 操作，该步骤会对 Try 预留的资源进行释放

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/4FAE4FD5607542D29F74D81F9DAFF41F/54287)

- XA 是资源层面的分布式事务，强一致性，在两阶段提交的整个过程中，一直会持有资源的锁
- TCC 是业务层面的分布式事务，最终一致性，不会一直持有资源的锁
  - TCC 是一种侵入式的分布式事务解决方案
  - 以上三个操作都需要业务系统自行实现，对业务系统有非常大的侵入性，设计相对复杂
  - **但优点是 TCC 完全不依赖数据库，能够实现跨数据库、跨应用资源管理**
  - 对这些不同数据访问通过侵入式的编码方式，实现一个原子操作，可以更好解决各种复杂业务场景下的分布式事务问题

### 以用户下单为例

#### try-commit

- try 阶段先进行预留资源，然后在 commit 阶段扣除资源，如下图：

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/2730B8B7E95447B59D8DB145C60E374F/55787)

#### try-cancel

- try 阶段先进行预留资源，中间有异常失败就导致全局事务回滚，在cancel 阶段释放资源，如下图：

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/768266383B6C44C28E6DEA98B784BF20/55786)

### Seata TCC模式

![](https://note.youdao.com/yws/public/resource/59f135645af771eb7c652f36dc0aae07/xmlnote/3D9BC147F5FA465BA15E705C0B61BBB9/55681)

- 在 Seata 中，AT 与 TCC 事实上都是两阶段提交的具体实现，它们的区别在于：
  - AT 模式基于支持本地 ACID 事务的关系型数据库
    - 一阶段 prepare 行为：
      - 在本地事务中，一并提交业务数据更新和相应回滚日志记录
    - 二阶段 commit 行为：
      - 马上成功结束，自动异步批量清理回滚日志
    - 二阶段 rollback 行为：
      - 通过回滚日志，自动生成补偿操作，完成数据回滚
  - TCC 模式不依赖于底层数据资源的事务支持
    - 一阶段 prepare 行为：
      - 调用自定义的 prepare 逻辑
    - 二阶段 commit 行为：
      - 调用自定义的 commit 逻辑
    - 二阶段 rollback 行为：
      - 调用自定义的 rollback 逻辑
- 简单来说，Seata的 TCC 模式就是手工的 AT 模式，允许用户自定义两阶段的处理逻辑，而不依赖 AT 模式的 undo_log

