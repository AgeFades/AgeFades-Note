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

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/0ED183D3FF244350B65E59CADDB7C5EF/17471)

##### 2. 建表(仅db模式)

- 全局事务会话信息由三项内容组成：
  - 全局事务：对应表 global_table
  - 分支事务：对应表 branch_table
  - 全局锁：对应表 lock_table
- 创建数据库 seata，执行 sql 脚本
  - srcipt/server/db/mysql.sql（seata源码sql脚本地址）

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/79042159C84E4825BB830F25E1726C93/17470)

##### 3. 修改store.mode

- 启动包：seata -> conf -> file.conf，修改 store.mode = "db"
- 源码：根目录 -> seata-server -> resources -> file.conf，修改 store.mode = "db"

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/B9AE178A9E6F4E989900B32E4DB106FD/17473)

##### 4. 修改数据库连接

- 启动包：seata -> conf -> file.conf，修改 store.db 相关属性
- 源码：根目录 -> seata-server -> resources -> file.conf，修改 store.db 相关属性

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/97178FFB1E8D43A89355A553A87F444A/17474)

##### 5. 配置Nacos注册中心

- 修改 conf 目录下的 registry.conf 配置
  - 将 seata-server 注册到 nacos 中

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/6D77A79278764D588DE0646E655425F3/17472)

##### 6. 配置Nacos配置中心

- 配置 Nacos 配置中心后，file.conf 可以不用配置
- 客户端配置 registry.conf 使用 nacos 时，
  - 要注意 group 和 seata server 中的 group 一致
  - 默认 gourp 是 `DEFAULT_GROUP`

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/634FF4BA22B74BD297FA61E8BBC61F40/17479)

- 修改 /seata/script/config-center/config.txt

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/A87F86DB907A4499B7DAE2DE341ABD30/17478)

- 配置事务分组，要与客户端的事务分组一致

```yaml
# 客户端事务分组配置
spring:
	cloud:
		alibaba:
			seata:
				tx-service-gourp: my_test_tx_group
```

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/249A70741E1B4909AF83BA07A4702AB4/17476)

- 配置参数同步到 nacos
  - /seata-server-1.4.0/seata/conf

```shell
sh nacos-config.sh -h localhost -p 8848 -u nacos -w nacos -g SEATA_GROUP -t 5a3c7d6c-f497-4d68-a71a-2e5e3340b3ca
```

- 参数说明
  - -h：host，默认值 localhost
  - -p：port，默认值 8848
  - -g：配置分组，默认值 `SEATA_GROUP`
  - -t：租户信息，对应 Nacos 的命名空间 id 字段，默认值为空``
- 精简配置

```apl
service.vgroupMapping.my_test_tx_group=default
service.default.grouplist=127.0.0.1:8091
service.enableDegrade=false
service.disableGlobalTransaction=false
store.mode=db
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true
store.db.user=root
store.db.password=root
store.db.minConn=5
store.db.maxConn=30
store.db.globalTable=global_table
store.db.branchTable=branch_table
store.db.queryLimit=100
store.db.lockTable=lock_table
store.db.maxWait=5000
```

##### 7. 启动Seata Server

- /seata-server-1.4.0/seata

```shell
 sh bin/seata-server.sh
```

- 支持的启动参数：
  - -h：指定在注册中心注册的 ip
    - 不指定时，获取当前ip
    - 部署在云环境或容器中的 server 建议指定
  - -p：server 启动的端口，默认 8091
  - -m：事务日志存储方式
    - file(默认)
    - db
    - redis：需 seata1.3版本及以上
  - -n：指定 seata-server 节点id
    - 如 1，2，3... 默认为 1
  - -e：指定 seata-server 运行环境
    - 如 dev，test...
    - 服务启动时会使用 `registry-dev.conf` 这样的配置

##### 启动成功

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/C3CEF164AF714B9EBBA794FAD3ECA801/17480)

![](https://note.youdao.com/yws/public/resource/798546ca0468451ad3e55de6407a9de4/xmlnote/CBB184544ADE4AE2A816ED56B5A64A00/17482)

