[TOC]

# Fox - MongoDB复制（副本）集实战及其原理分析

- [复制集视屏搭建教程](https://vip.tulingxueyuan.cn/detail/p_622d92aee4b066e9608ee2c9/6)。
- 如果想快速搭建复制集测试环境，可以使用mtools工具

## 复制集架构

- 生产环境中，不建议使用单机版的MongoDB
  - 单机版无法保证可靠性，一旦服务宕机或故障，业务就直接不可用
  - 一旦服务磁盘损坏，数据就直接丢失
- MongoDB复制集（Replication Set）由一组Mongod实例组成
  - 包含**一个 Primary 节点**和 **多个Secondary节点**
- MongoDB Driver（客户端）的所有数据都写入 Primary
  - Secondary 从 Primary 同步写入的数据，以提供数据高可用
  - **复制集提供冗余和高可用，是所有生产部署的基础**
- 实现基于两个方面的功能：
  1. 在数据写入时，将数据迅速复制到另一个独立节点上
  2. 在接受写入的节点发生故障时，自动选举出一个新的替代节点

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/78DF2927AF9542C084EBD6695948DB98/40801)

- 在实现高可用的同事，复制集实现了其他几个附加作用：
  - **数据分发**：将数据从一个区域复制到另一个区域，减少另一个区域的读延迟
  - **读写分离**：不同类型的压力分别在不同的节点上执行
  - **异地容灾**：在数据中心故障时，迅速切换到异地
- 早期的MongoDB使用的是一种 Master-Slave 的架构，在3.4版本后被废弃

### 三节点复制集模式

- 常见的复制集架构由3个成员节点组成，其中存在几种不同的模式。

#### PSS模式（官方推荐模式）

- PSS即：一个主节点 + 两个备节点

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/1FA654A15B3D4E52B87568B3B55A9767/40950)

- 此模式始终提供数据集的两个完整副本，如果主节点不可用，则复制集选择备节点作为主节点并继续正常工作。旧的主节点在可用时，重新加入复制集。

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/391AA5BDA402457388ADC0721417EF00/40951)

#### PSA模式

- 由一个主节点、一个备节点和一个仲裁节点（Arbiter）组成

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/77B8762B7E674B6E8DB73BE3B0EA754E/40949)

- 其中，Arbiter节点不存储数据副本，也不提供业务的读写操作。
  - Arbiter节点发生故障时，不影响业务，只影响选举投票。
- 此模式仅提供数据的一个完整副本，如果主节点不可用，则复制集将选择备节点作为主节点

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/7AE207634AE542FF832E634EEC0A7F70/40952)

## 典型三节点复制集环境搭建

- 即使暂时只有一台服务器，也要以单节点模式启动复制集
  - 单机多实例启动复制集
  - 单节点启动复制集

### 复制集注意事项

#### 关于硬件

- 因为正常的复制集节点都有可能成为主节点，所以硬件配置上尽量保持一致
- 为了保证节点不会同时宕机，各节点使用的硬件必须具有独立性

#### 关于软件

- 复制集各节点版本必须一致，避免出现不可预知的问题
- 增加节点不会增加系统写性能

### 准备配置文件

- 复制集的每个mongod进程应该位于不同的服务器，现在演示一台机器上运行三个进程，要各自配置

  - 不同的端口（28017/28018/28019）

  - 不同的数据目录

    - ```shell
      mkdir -p /data/db{1,2,3}
      ```

  - 不同日志文件路径（例如：/data/db1/mongod.log）

- 创建配置文件（yaml格式）、不同节点修改 文件路径、节点端口号

```shell
vim /data/db1/mongod.conf
```

```yaml
systemLog:
  destination: file
  path: /data/db1/mongod.log # log path
  logAppend: true
storage:   
  dbPath: /data/db1 # data directory      
net:
  bindIp: 0.0.0.0
  port: 28017 # port
replication:
  replSetName: rs0  
processManagement:
  fork: true
```

### 启动MongoDB进程

```shell
mongod -f /data/db1/mongod.conf
mongod -f /data/db2/mongod.conf
mongod -f /data/db3/mongod.conf
```

- 注意：如果启用了 SELinux，可能阻止上述进程启动。简单处理就是关闭 SELinux

```shell
# 永久关闭,将SELINUX=enforcing改为SELINUX=disabled,设置后需要重启才能生效
vim /etc/selinux/config

# 查看SELINUX
/usr/sbin/sestatus -v
```

### 配置复制集

- 复制集通过 **replSetInitiate** 命令或 **mongo shell的initiate()** 进行初始化
  - 初始化后，各个成员间开始发送心跳消息，并发起Priamry选举操作
  - 获得多数成员投票支持的节点，会成为Primary，其余的为Secondary

```shell
# 方式1

# mongo --port 28017
# 初始化复制集
> rs.initiate() 
# 将其余成员添加到复制集
> rs.add("192.168.65.174:28018") 
> rs.add("192.168.65.174:28019")
```

```shell
# 方式2

# mongo --port 28017 
# 初始化复制集
> rs.initiate({
    _id: "rs0",
    members: [{
        _id: 0,
        host: "192.168.65.174:28017"
    },{
        _id: 1,
        host: "192.168.65.174:28018"
    },{
        _id: 2,
        host: "192.168.65.174:28019"
    }]
})
```

### 验证

- MongoDB主节点进行写入

```shell
# mongo --port 28017
rs0:PRIMARY> db.user.insert([{name:"fox"},{name:"monkey"}])
```

- MongoDB从节点进行读

```shell
# mongo --port 28018
# 指定从节点可读
rs0:SECONDARY> rs.secondaryOk()
rs0:SECONDARY> db.user.find()
```

### 复制集状态查询

- 查看复制集整体状态
  - 可查看各成员当前状态，包含是否健康、是否在全量同步、心跳信息、增量同步信息、选举信息、上一次的心跳时间等

```apl
rs.status()
```

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/F6AD92ACE20A43AE997F42BD064D70D1/41295)

- members 一列体现了所有复制集成员的状态，主要如下：
  - health：成员是否健康，通过心跳进行检测
  - state/stateStr：成员的状态，PRIMARY表述主节点，SECONDARY表示备节点，如果节点出现故障，可能出现一些其他的状态，例如：RECOVERY
  - uptime：成员的启动时间
  - optime/optimeDate：成员最后一条同步oplog的时间
  - pingMs：成员与当前节点的ping时延
  - syncingTo：成员的同步来源
- 查看当前节点角色：
  - 除了当前节点信息，是一个更精简化的信息，也返回整个复制集的成员列表，真正的Primary是谁，协议相关的配置信息等，Driver在首次连接复制集时会发送该命令

```apl
db.isMaster()
```

### Mongo Shell复制集命令

| **命令**                           | **描述**                                                 |
| ---------------------------------- | -------------------------------------------------------- |
| rs.add()                           | 为复制集新增节点                                         |
| rs.addArb()                        | 为复制集新增一个 arbiter                                 |
| rs.conf()                          | 返回复制集配置信息                                       |
| rs.freeze()                        | 防止当前节点在一段时间内选举成为主节点                   |
| rs.help()                          | 返回 replica set 的命令帮助                              |
| rs.initiate()                      | 初始化一个新的复制集                                     |
| rs.printReplicationInfo()          | 以主节点的视角返回复制的状态报告                         |
| rs.printSecondaryReplicationInfo() | 以从节点的视角返回复制状态报告                           |
| rs.reconfig()                      | 通过重新应用复制集配置来为复制集更新配置                 |
| rs.remove()                        | 从复制集中移除一个节点                                   |
| rs.secondaryOk()                   | 为当前的连接设置 从节点可读                              |
| rs.status()                        | 返回复制集状态信息。                                     |
| rs.stepDown()                      | 让当前的 primary 变为从节点并触发 election               |
| rs.syncFrom()                      | 设置复制集节点从哪个节点处同步数据，将会覆盖默认选取逻辑 |

## 使用mtools搭建复制集

- [mtools](http://blog.rueckstiess.com/mtools/)是一套基于Python实现的MongoDB工具集，包括 日志分析、报表生成、简易的数据库安装等功能。
- mtools所包含的一些常用组件如下：

| Tools        | Description                                          |
| ------------ | ---------------------------------------------------- |
| mlogfilter   | 合并、分割日志文件，过滤慢查询，集合扫描，格式转换等 |
| mloginfo     | 统计日志内的数据库信息（启停、连接、集群状态等)      |
| mplotqueries | 日志转化为图表形式                                   |
| mlogvis      | 日志转化为HTML页面，与mplotqueries类似               |
| mlaunch      | 快速搭建本地测试环境(单机、集群、分片)               |

### 安装mtools

#### 环境准备

- mtools需要调用MongoDB的二进制程序来启动数据库，因此需保证Path路径中包含{MONGODB_HOME}/bin 这个目录
- 需要安装Python环境，需选用Python3.7、3.8、3.9版本

##### Centos7安装Python3.9

###### 1. 安装编译相关工具

```shell
yum -y groupinstall "Development tools"
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
yum install libffi-devel -y
```

###### 2. 下载安装包解压

```shell
wget https://www.python.org/ftp/python/3.9.8/Python-3.9.8.tgz
tar -xvJf  Python-3.9.8.tar.xz
```

###### 3. 编译安装

```shell
# 创建编译安装目录
mkdir /usr/local/python3 
cd Python-3.9.8
./configure --prefix=/usr/python3
make && make install
```

###### 4. 创建软连接

```shell
ln -s /usr/local/python3/bin/python3 /usr/bin/python3
ln -s /usr/local/python3/bin/pip3 /usr/bin/pip3
```

###### 5. 验证

```shell
python3 -V
pip3 -V
```

![](https://note.youdao.com/yws/public/resource/476edebe424c4d48bc433b5cd88df323/xmlnote/11CD55BC9A2D4C6A9E51DF116FCDB4B6/42195)

#### pip安装

- 安装依赖

```shell
pip3 install python-dateutil
pip3 install psutil pymongo 
```

- 安装mtools

```shell
pip3 install mtools
```

#### 通过源码安装

```shell
wget https://github.com/rueckstiess/mtools/archive/refs/tags/v1.6.4.tar.gz
# 解压后进入mtools
python setup.py install
```

### mtools创建复制集

```shell
# 准备复制集使用的工作目录
mkdir -p /data/mongo
cd /data/mongo
# 初始化3节点复制集
mlaunch init --replicaset --nodes 3
```

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/04D49FA38C4D43308AD79AD7FC120E93/42325)

- 查看复制集状态

```shell
 mongo --port 27017
 replset:PRIMARY> rs.status()
```

### mtools创建分片集群

```shell
# 准备分片集群使用的工作目录
mkdir /data/mongo-cluster
cd /data/mongo-cluster/
# 执行mlaunch init初始化集群
mlaunch init --sharded 2 --replicaset --node 3 --config 3 --csrs --mongos 3 --port 27050 
```

- 选项说明：
  - --sharded 2：启用分片集群模式，分片数为2。
  -  --replicaset --nodes 3：采用3节点的复制集架构，即每个分片为一致的复制集模式。
  -  --config 3 --csrs：配置服务器采用3节点的复制集架构模式，--csrs是指Config Server as a Replica Set
  -  --mongos 3：启动3个mongos实例进程。
  - --port 27050：集群将以27050作为起始端口，集群中的各个实例基于该端口向上递增。
  -  --noauth：不启用鉴权。
  - --arbiter 向复制集中添加一个额外的仲裁器
  - --single 创建单个独立节点
  - --dir 数据目录，默认是./data
  - --binarypath 如果环境有二进制文件，则不用指定

- 执行成功的输出

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/A21D7E6FE48E406D8216582286E18886/42307)

#### 检查分片实例

- mlaunch list 命令可以对当前集群的实例状态进行检查
  - 可以看到各个实例运行状态，包括进程号以及监听的端口号等

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/C33C0550ACFE4C7DA43028D702728C81/42268)

```shell
# 显示标签
mlaunch  list --tags 
# 显示启动命令
mlaunch  list --startup
```

#### 连接mongos，查看分片实例的情况

```shell
mongo --port 27050
mongos> db.adminCommand({listShards:1})
```

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/E5F01457096743C2AF0673575DD74319/42275)

#### 停止、启动

- mlaunch stop、mlaunch start

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/BFA11271186841329E65394BFC563CC5/42281)

![](https://note.youdao.com/yws/public/resource/3c02251c8b4a8bfc98ab392146aa8222/xmlnote/062DEB3872444B07B65B556135F353E9/42284)

## 安全认证

### 创建用户

- 在主节点服务器上，启动mongo

```apl
use admin
# 创建用户
db.createUser( {
    user: "fox",
    pwd: "fox",
     roles: [ { role: "clusterAdmin", db: "admin" } ,
         { role: "userAdminAnyDatabase", db: "admin"},
         { role: "userAdminAnyDatabase", db: "admin"},
         { role: "readWriteAnyDatabase", db: "admin"}]
})
```

### 创建keyFile文件

- keyFile文件用于 集群之间的安全认证（开启keyFile认证就默认开启了auth认证了）
- 注意：创建keyFile前，需要先停掉复制集中所有主从节点的mongod服务，然后再创建，否则可能出现服务启动不了的情况
- 将主节点中的keyFile文件拷贝到复制集其他从节点服务器中，路径地址对应mongo.conf配置文件中的keyFile字段地址，并设置keyFile权限为600

```apl
# mongo.key采用随机算法生成，用作节点内部通信的密钥文件。
openssl rand -base64 756 > /data/mongo.key
 # 权限必须是600
 chmod 600 /data/mongo.key  
```

### 测试

```shell
# 进入主节点
mongo --port 28017
```

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/E3265ED0F4584F609F618E328D0B9799/41111)

```shell
# 进入主节点
mongo --port 28017 -ufox -pfox --authenticationDatabase=admin
```

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/62F6F73123A94AB5B01B8F7649114428/41112)

## 复制集连接方式

- 方式一：直接连接Primary节点，正常可读写，主节点故障时，无法正常访问

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/495ACBAAECD147C6905F45C5684BDE9E/40973)

- 方式二（强烈推荐）：通过高可用Uri的方式连接，主节点故障时，MongoDB Driver可自动感知并切换流量路由到新的 Primary 节点

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/839B848AA23F485595517F3ECDDB755B/40978)

### SpringBoot操作复制集配置

```yaml
spring:
  data:
    mongodb:
        uri: mongodb://fox:fox@192.168.65.174:28017,192.168.65.174:28018,192.168.65.174:28019/test?authSource=admin&replicaSet=rs0
```

## 复制集成员角色

- 复制集里面有多个节点，每个节点拥有不同的职责
  - 看成员角色之前，需要先了解以下两个重要属性

### 两个重要属性

#### 1. Priority = 0

- **当Priority = 0时，它不可以被复制集选举为主**
  - Priority的值越高，被选举为主的概率就越大。

#### 2. Vote = 0

- **不可以参与选举投票，此时该节点的Priority也必须为0，即它也不能被选举为主**
- 由于一个复制集中最多只有7个投票成员，因此多出来的成员则必须将其 vote 设置为 0

### 成员角色

#### Primary

- 主节点，接收所有写请求，然后将修改同步到所有备节点
  - 一个复制集只有一个主节点，宕机后，其他节点重新推选一个新的主节点

#### Secondary

- 备节点，与主节点保持同样的数据集，主节点宕机后，参与竞选主节点
- 分为以下三个不同类型：
  - Hidden = false：正常的只读节点，是否可选为主、可参与选举，取决于Priroty、Vote的值
  - Hidden = true：隐藏节点，对客户端不可见，可以参与选举，但是Priority必须为0，即不能被推选为主节点
    - 由于隐藏节点不会接收业务访问，因此可通过隐藏节点做一些数据备份、离线计算的任务，不会影响整个复制集
  - Delayed：延迟节点，必须同时具备隐藏节点和 Priority0的特性，会延迟一定的时间（SlaveDelay配置决定）从上游复制增量，常用于快速回滚场景。

#### Arbiter

- 仲裁节点，只用于参与选举投票，本身不承载任何数据，只作为投票角色
  - 当复制集节点为偶数时，最好加入一个Arbiter节点（不然偶数节点投票可能出现过不了半数的场景）

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/48FA2F4D6517485C9B6CBBB6F994A2CF/41168)

### 配置隐藏节点

- 大部分情况下，将节点设置为隐藏节点是用来协助 delayed members 的
  - 如果仅需要防止该节点成为主节点，设置其 priority 0 member 即可实现

```apl
cfg = rs.conf()
cfg.members[1].priority = 0
cfg.members[1].hidden = true
rs.reconfig(cfg)
```

- 设置完毕后，该从节点的优先级将变为0来防止其升级为主节点
  - 同时对其应用程序也是不可见的，在其他节点上执行 db.isMaster() 将不会显示隐藏节点

### 配置延时节点

- 配置一个延时节点时，复制过程与该节点的 oplog 都将延时，延时节点中的数据集将会比复制集主节点中的数据延后。
  - 举例：如果延时1h，当前10点，那延时节点的数据集中就不会有9点后的操作

```apl
cfg = rs.conf()
cfg.members[1].priority = 0
cfg.members[1].hidden = true
# 延迟1分钟
cfg.members[1].slaveDelay = 60
rs.reconfig(cfg)
```

#### 查看复制延迟

- 想查看当前节点oplog的情况，可以用 **rs.printReplicationInfo**() 命令

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/C99764E938A34A8DACB654016ECC608A/41299)

- 描述了 oplog 的大小、最早一条oplog及最后一条oplog的产生时间
  - log length start to end 指的是一个 复制窗口（时间差）
  - **通常oplog大小不变时，业务写操作越频繁，复制窗口就越短**
- 在节点上执行 **rs.printSecondaryReplicationInfo()** 可以一并列入所有备节点的同步延迟情况

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/7022C2353C57434DB82BF4A744A13DA2/41303)

### 添加投票节点

```apl
# 为仲裁节点创建数据目录，存放配置数据。该目录将不保存数据集
mkdir /data/arb
# 启动仲裁节点，指定数据目录和复制集名称
mongod --port 30000 --dbpath /data/arb --replSet rs0 
# 进入mongo shell,添加仲裁节点到复制集
rs.addArb("ip:30000")
```

### 移除复制集节点

```apl
# 1.关闭节点实例
# 2.连接主节点，执行下面命令
rs.remove("ip:port")
```

或者

```apl
# 1.关闭节点实例
# 2.连接主节点，执行下面命令
cfg = rs.conf()
cfg.members.splice(2,1)  #从2开始移除1个元素
rs.reconfig(cfg)
```

### 更改复制集节点

```apl
cfg = rs.conf()
cfg.members[0].host = "ip:port"
rs.reconfig(cfg)
```

## 复制集高可用

### 复制集选举

- MongoDB的复制集选举使用 [Raft算法](https://raft.github.io/) 来实现，**选举成功的必要条件是大多数投票节点存活**。
- 具体实现中，MongoDB对Raft协议做了一扩展，包括：
  - 支持 chainingAllowed 链式复制，即 **备节点不只是从主节点上同步数据，还可以选择一个离自己最近（心跳延时最小）的节点来复制数据**
  - 增加了预投票机制，即 preVote，这主要是用来避免网络分区时产生Term（任期）值激增的问题
  - **支持投票优先级**，如果备节点发现自己的优先级比主节点高，则会主动发起投票并尝试称为新的主节点
- **一个复制集最多可以有50个成员，但只有7个投票成员**
  - 这是因为一旦过多的成员参与数据复制、投票过程，将会带来更多可靠性方面的问题

| 投票成员数 | 大多数 | 容忍失效数 |
| ---------- | ------ | ---------- |
| 1          | 1      | 0          |
| 2          | 2      | 0          |
| 3          | 2      | 1          |
| 4          | 3      | 1          |
| 5          | 3      | 2          |
| 6          | 4      | 2          |
| 7          | 4      | 3          |

- **当复制集内存活的成员数量不足大多数时，整个复制集将无法选举出主节点，此时无法提供写服务，这些节点将只处于只读状态**
- 此外，如果希望避免平票结果的产生，最好使用奇数个节点成员
  - 在MongoDB复制集的实现中，对于平票问题也提供了解决方案：
    - 为选举定时器增加少量的随机时间偏差，避免各个节点在同一时刻发起选举，提高成功率
    - 使用仲裁者角色，该角色不做数据复制，也不承担读写业务，仅仅用来投票

### 自动故障转移

- 在故障转移场景中，比较关心的问题是：
  - **备节点如何感知主节点故障的？**
  - **如果降低故障转移对业务产生的影响？**

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/B60959404DAC42CF98FBAD74BD6ED281/41646)

- **一个影响检测机制的因素是心跳，在复制集组建完成之后，各成员节点会开启定时器，持续向其他成员发起心跳**
  - 参数 heartbeatIntervalMillis（心跳间隔时间），默认是2s。
    - 如果心跳成功，则会持续以2s的频率继续发送心跳；
    - 如果心跳失败，则会立即重试心跳，一直到心跳恢复成功
- **另一个重要因素是 选举超时检测，一次心跳检测失败并不会立即触发重新选举**
  - 实际上除了心跳，成员节点还会开启一个 选举超时检测定时器，该定时器默认以10s的间隔执行，具体可以通过 electionTimeoutMillis 参数指定
    - 如果心跳成功，则取消上一次的 electionTimeout 调度（保证不会发起选举），并发起新一轮 electionTimeout 调度
    - 如果心跳响应一直不成功，则 electionTimeout 任务触发，进而导致备节点发起选举并称为新的主节点
- Mongo的实现中，选举超时检测的周期要略大于 electionTimeoutMillis 设定，会加入一个随机偏移量，大约在 10 ~ 11.5s
  - 这样设计是为了错开多个备节点主动发起选举的时间，提高选举成功率
- **因此，在electionTimeout任务中触发选举，必须要满足如下条件**
  1. 当前节点是备节点
  2. 当前节点具备选举权限
  3. 在检测周期内仍然没有与主节点心跳成功

#### 业务影响评估

- **在复制集发生主备节点切换的情况下，会出现短暂的无主节点阶段，此时无法接受业务写操作**
  - 如果是因为主节点故障导致的切换，则对该节点的所有读写操作都会延时
  - Mongo3.6及以上版本，可以通过开启 retryWrite 来降低影响

```apl
# MongoDB Drivers 启用可重试写入
mongodb://localhost/?retryWrites=true
# mongo shell
mongo --retryWrites
```

- **如果主节点属于强制掉电，那整个Failover过程将会变长**
  - 很可能在 Election 定时器超时后，才能被其他节点感知并恢复，这个时间窗口一般在 12s 以内
  - 然而实际上，还应该考量客户端或mongos对于复制集角色的监视和感知行为（真实的情况可能需要30s以上）
  - 对于非常重要的业务，建议在业务层面做一些防护策略，比如设计重试机制。
- **如果优雅的重启复制集？**
  1. 逐个重启复制集里所有 Secondary 节点
  2. 对Primary发送 rs.stepDown() 命令，等待 primary 降级为 Secondary
  3. 重启降级后的 Primary

### 复制集数据同步机制

- **在复制集架构中，主节点与备节点之间是通过oplog来同步数据的**
  - 这里的 oplog 是一个特殊的固定集合，当主节点上的一个写操作完成后，会向oplog集合写入一条对应的日志
  - 而备节点则通过这个oplog不断拉取到新的日志，在本地进行回放，已达到数据同步的目的

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/F51C3FFD192B435FBD3E58D18CB75F61/41360)

#### 什么是oplog

- **MongoDB oplog 是 Local 库下的一个集合，用来保存写操作所产生的增量日志**（类似于Mysql的 binlog）
- **是一个 Capped Collection（固定集合）**
  - 即：超出配置的最大值后，会自动删除最老的历史数据，MongoDB针对 oplog 的删除有特殊优化，以提升删除效率
- 主节点产生新的 oplog entry，从节点通过复制 oplog 并应用，来保持和主节点的状态一致

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/B1F19FCDF27C414D812753F0179732C4/41351)

#### 查看oplog

```apl
use local
db.oplog.rs.find().sort({$natural:-1}).pretty()
```

- local.system.replset：用来记录当前复制集的成员
- local.startup_log：用来记录本地数据库的启动日志信息
- local.replset.minvalid：用来记录复制集的跟踪信息，如初始化同步需要的字段

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/9CAAFE293BAD482285C81FF1A5BEC4CF/43744)

- ts：操作时间，当前timestamp + 计数器，计数器每秒都被重置
- v：oplog版本信息
- op：操作类型
  - i：插入
  - u：更新
  - d：删除
  - c：执行命令（如 createDatabase、dropDatabase）
  - n：空操作，特殊用途
- ns：操作针对的集合
- o：操作内容
- o2：操作查询条件，仅update操作包含该字段

##### ts

- ts字段描述了oplog产生的时间戳，可称之为optime。
  - **optime是备节点实现增量日志同步的关键**，保证了oplog是节点有序的，由两部分组成：
    - 当前的系统时间，即UNIX时间至现在的描述，32位
    - 整数计时器，不同时间值会将计数器进行重置，32位
- optime属于BSON的Timestamp类型，既然 **oplog保证了节点有序，那么备节点便可以通过轮训的方式进行拉取**
  - 这里用到了 可持续追踪的游标（tailable cursor） 技术

![](https://note.youdao.com/yws/public/resource/768f4ec76d7d637bae9abe826958a78f/xmlnote/B789A36D5C374AF0A8DE2A4A8D1FA17E/41436)

- **每个备节点都分别维护了自己的offset，也就是从主节点拉取的最后一条日志的optime，在执行同步时就通过这个optime向主节点的oplog集合发起查询**
  - 为了避免不停地发起新的查询连接，在启动第一次查询后，可以将 cursor 挂住（通过将 cursor 设置为 tailable）
  - 这样只要oplog中产生了新的记录，备节点就能使用同个请求通道获取数据
  - taiable cursor只有在查询的集合为固定集合时，才允许开启

#### oplog集合的大小

- 可通过参数 replication.oplogSizeMB 设置，对于64位系统来说，oplog的默认值为：

```apl
oplogSizeMB = min(磁盘可用空间*5%，50GB)
```

- 大多数业务场景下，很难一开始就评估出一个合适的 oplogSize
  - MongoDB4.0版本之后，提供了 **replSetResizeOplog** 命令，可以不重启动态修改 oplogSize

```apl
# 将复制集成员的oplog大小修改为60g  指定大小必须大于990M
db.adminCommand({replSetResizeOplog: 1, size: 60000})
# 查看oplog大小
use local
db.oplog.rs.stats().maxSize
```

#### 幂等性

- 每一条oplog记录都描述了一次数据的原子性变更，**对于oplog来说，必须保证是幂等性的**。
  - 即：对同一个 oplog，无论进行多少次回放操作，数据的最终状态都是一致的。
  - 举例：某文档x字段当前值为100，用户向 Primary 发送一条 {$inc: {x:1}}，记录oplog时，会将其转化成 {$set: {x:101}}，才能保证幂等性

##### 幂等性的代价

- 简单元素操作，$inc转为$set没什么影响，执行开销也差不多，但遇到数组元素操作时，就不一样了
- 测试：

```apl
db.coll.insert({_id:1,x:[1,2,3]})
```

- 在数组尾部push 2个元素，查看oplog发现 $push 被转换成了 $set 操作（设定数组指定位置的元素为某个值）

```apl
rs0:PRIMARY> db.coll.update({_id: 1}, {$push: {x: { $each: [4, 5] }}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
rs0:PRIMARY> db.coll.find()
{ "_id" : 1, "x" : [ 1, 2, 3, 4, 5 ] }
rs0:PRIMARY> use local
switched to db local
rs0:PRIMARY> db.oplog.rs.find({ns:"test.coll"}).sort({$natural:-1}).pretty()
{
	"op" : "u",
	"ns" : "test.coll",
	"ui" : UUID("69c871e8-8f99-4734-be5f-c9c5d8565198"),
	"o" : {
		"$v" : 1,
		"$set" : {
			"x.3" : 4,
			"x.4" : 5
		}
	},
	"o2" : {
		"_id" : 1
	},
	"ts" : Timestamp(1646223051, 1),
	"t" : NumberLong(4),
	"v" : NumberLong(2),
	"wall" : ISODate("2022-03-02T12:10:51.882Z")
}
```

- $push转为带具体位置的$set开销差不多，但看看往数组头插两个元素试试

```apl
rs0:PRIMARY> use test
switched to db test
rs0:PRIMARY> db.coll.update({_id: 1}, {$push: {x: { $each: [6, 7], $position: 0 }}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
rs0:PRIMARY> db.coll.find()
{ "_id" : 1, "x" : [ 6, 7, 1, 2, 3, 4, 5 ] }
rs0:PRIMARY> use local
switched to db local
rs0:PRIMARY> db.oplog.rs.find({ns:"test.coll"}).sort({$natural:-1}).pretty()
{
	"op" : "u",
	"ns" : "test.coll",
	"ui" : UUID("69c871e8-8f99-4734-be5f-c9c5d8565198"),
	"o" : {
		"$v" : 1,
		"$set" : {
			"x" : [
				6,
				7,
				1,
				2,
				3,
				4,
				5
			]
		}
	},
	"o2" : {
		"_id" : 1
	},
	"ts" : Timestamp(1646223232, 1),
	"t" : NumberLong(4),
	"v" : NumberLong(2),
	"wall" : ISODate("2022-03-02T12:13:52.076Z")
}
```

- 可以看出，**向数组头插数据时**，oplpg里的 $set 不再是设置某个位置的值（因为所有元素位置都发生了变化），而是 $set 数组最终的结果（即整个数组内容都要写入oplog）
  - 当$push指定了$slice或者$sort参数时，oplog也会将整个数组内容作为 $set 的参数
  - $pull、$addToSet 等更新操作也是类似，这样才能保证幂等性
- **oplog的写入被放大，导致同步追不上 --- 大数组更新**
  - 当数组非常大时，对数组的一个小更新，可能就需要把整个数组的内容都记录到 oplog 里。
  - 举例：某个集合文档中有一个很大的数组字段，比如说1000个元素、大小为64kb左右，该数组元素按时间反序存储，新插入的元素放在第一位，然后保留数组的前1000个元素（$slice:1000）
  - 上述场景中，Primary上每次往该数组插入一个新元素（请求数大概几百字节），oplog就要记录整个数组的内容，Secondary同步时会拉取oplog并重放，Primary到Secondary同步oplog的流量，是客户端到Primary流量的几百倍
    - 导致主备间网卡流量跑满，而且由于oplog的量太大，旧的内容很快就删除掉，最终导致Secondary追不上，就转换成了 RECOVERING 状态。
- 文档中使用数组，一定要注意上述问题，尽量注意：
  1. 数组元素个数不要太多，总的大小也不要太大
  2. 尽量避免对数组进行更新操作
  3. **如果一定要更新，尽量只在尾部插入元素，复杂的逻辑可以考虑在业务层面上来支持**

#### 复制延迟（replication lag）

- 由于oplog集合是有固定大小的，因此存放在里面的oplog随时可能会被新的记录冲掉
- **如果备节点的复制不够快，就无法跟上主节点，从而产生复制延迟问题**
  - 一旦备节点的延迟过大，则随时可能发生复制断裂的风险，这意味着备节点的optime(最新一条同步记录)已经被主节点老化掉，备节点就无法继续进行数据同步
- 为了尽量避免复制延迟带来的风险，可以采用如下措施：
  - **增加oplog的容量大小，并保持对复制窗口的监视**
  - 通过一些扩展手段降低主节点的写入速度
  - 优化主备节点之间的网络
  - **避免字段使用太大的数组（可能导致oplog膨胀）**

#### 数据回滚

- 由于复制延迟是不可避免的，这意味着 主备节点间的数据无法保持绝对的同步。
- **当主节点宕机时，备节点被选举为主节点，当旧的主节点重新加入时，必须回滚掉之前的一些“脏日志数据”，以保证数据集与新的主节点一致**
  - 主备复制集合的差距越大，发生大量数据回滚的风险就越高
- **对于写入的业务数据来说，如果已经被复制到了复制集的大多数节点，则可以避免被回滚的风险**
  - 应用上可以通过设定更高的写入级别（writeConcern：majority）来保证数据的持久性。
  - 这些由旧主节点回滚的数据会被写到单独的 rollback 目录下，必要情况下仍然可以恢复这些数据。
- 当rollback发生时，MongoDB将这些数据以 BSON 形式存放到 dbpath 路径下 rollback文件夹中
  - BSON文件的命名格式：<database>.<collection>.<timestamp>.bson

```apl
mongorestore --host 192.168.192:27018 --db test --collection emp -ufox -pfox 
--authenticationDatabase=admin rollback/emp_rollback.bson
```

#### 同步源选择

- MongoDB允许通过备节点进行复制，会发生在如下情况中：
  - **在setting.chainingAllowed开启（默认开启）的情况下，备节点自动选择一个最近的节点（ping命令时延最小）进行同步**
- 可以如下操作关闭 setting.chaningAllowed

```apl
cfg = rs.config()
cfg.settings.chainingAllowed = false
rs.reconfig（cfg)
```

- 使用 replSetSyncFrom 命令临时更改当前节点的同步源，比如在初始化同步时，将同步源指向备节点，来降低对主节点的影响

```apl
db.adminCommand( { replSetSyncFrom: "hostname:port" })
```

