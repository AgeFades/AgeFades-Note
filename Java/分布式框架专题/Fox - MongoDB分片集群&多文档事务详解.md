[TOC]

# Fox - MongoDB分片集群&多文档事务详解

## 分片集群架构

### 分片简介

- **分片（shard）是指在数据库进行水平切分之后，将其存储到多个不同的服务器节点上的一种扩展方式**
  - 概念类似 "水平分表"，MongoDB本身就自带分片管理能力

### 为什么要使用分片

- **MongoDB复制集实现了数据的多副本复制及高可用，但是一个复制集能承载的容量和负载是有限的**
- 遇到如下场景时，就该考虑使用分片了：
  - 存储容量需求超出单机的磁盘容量
  - 活跃的数据集超出单机内存容量，导致很多请求都要从磁盘读取数据，影响性能
  - 写IO ps超出单个MongoDB节点的写服务能力

### MongDB分片集群架构

- Sharded Cluster 是对数据进行水平扩展的一种方式。**MongoDB使用分片集群来支持大数据集和高吞吐量的业务场景**
- 在分片模式下，存储不同的切片数据的节点被称为分片节点，一个分片集群内包含了多个分片节点。
  - 除了分片节点，集群中还需要一些配置节点、路由节点，以保证分片机制的正常运作

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/D4BD94A9017E43BAB926B3E1F6D73EBA/41963)

#### 核心概念

##### 数据分片

- 用于存储真正的数据，并提供最终的数据读写访问。
- 分片仅仅是一个逻辑概念，可以是一个单独的mongod实例，也可以是一个复制集
  - 图中的Shard1、Shard2都是一个复制集分片
- **在生产环境中也一般会使用复制集的方式，这是为了防止数据节点出现单点故障**

##### 查询路由

- mongos：**是分片集群的访问入口，其本身并不持久化数据**
- mongos启动后，会从配置服务器中加载元数据。
  - 之后mongos开始提供访问服务，并将用户的请求正确路由到对应的分片
  - 在分片集群中可以部署多个mongos以分担客户端请求的压力

### 环境搭建

[搭建视屏](https://vip.tulingxueyuan.cn/detail/p_622d92aee4b066e9608ee2c9/6)

- 这里演示正常搭建（实际可以使用mtools搭建，更快更简单，参考上一篇内容）

#### 环境准备

- 3台Linux虚拟机，准备MongoDB环境，配置环境变量。
- 一定要版本一致（重点），当前使用 version4.4.9   

![](https://note.youdao.com/yws/public/resource/fa7b18a78fa8a4d1961729b043388b57/xmlnote/8544839491354BEAB63D82BC86250793/35497)

#### 配置域名解析

- 在三台服务器上执行如下命令，注意替换实际ip地址

```shell
echo "192.168.65.97  mongo1 mongo01.com mongo02.com" >> /etc/hosts
echo "192.168.65.190 mongo2 mongo03.com mongo04.com" >> /etc/hosts
echo "192.168.65.200 mongo3 mongo05.com mongo06.com" >> /etc/hosts 
```

#### 准备分片目录

- 在各服务器上创建数据目录，这里使用 /data，可自行更改
- 在mongo01.com / mongo03.com / mongo05.com 执行

```shell
mkdir -p /data/shard1/db  /data/shard1/log   /data/config/db  /data/config/log
```

- 在mongo02.com / mongo04.com / mongo06.com 上执行以下命令： 

```shell
mkdir -p /data/shard2/db  /data/shard2/log   /data/mongos/
```

#### 创建第一个分片用的复制集

- 在mongo01.com / mongo03.com / mongo05.com 上执行以下命令：
  - --shardsvr：声明这是一个集群的分片
  - --wiredTigerCacheSizeGB：设置内存大小

```shell
mongod --bind_ip 0.0.0.0 --replSet shard1 --dbpath /data/shard1/db \
--logpath /data/shard1/log/mongod.log --port 27010 --fork \
--shardsvr --wiredTigerCacheSizeGB 1
```

#### 初始化第一个分片复制集

```apl
# 进入mongo shell
mongo mongo01.com:27010
# shard1复制集节点初始化
rs.initiate({
    _id: "shard1",
    "members" : [
    {
        "_id": 0,
        "host" : "mongo01.com:27010"
    },
    {
        "_id": 1,
        "host" : "mongo03.com:27010"
    },
    {
        "_id": 2,
        "host" : "mongo05.com:27010"
    }
    ]
})
# 查看复制集状态
rs.status()
```

![](https://note.youdao.com/yws/public/resource/fa7b18a78fa8a4d1961729b043388b57/xmlnote/3F8B78A7557D4BF0BAB50C0A60D9908A/35378)

#### 创建config server复制集

- 在mongo01.com / mongo03.com / mongo05.com上执行以下命令： 

```apl
mongod --bind_ip 0.0.0.0 --replSet config --dbpath /data/config/db \
--logpath /data/config/log/mongod.log --port 27019 --fork \
--configsvr --wiredTigerCacheSizeGB 1
```

#### 初始化config server复制集

```apl
# 进入mongo shell
mongo mongo01.com:27019
# config复制集节点初始化
rs.initiate({
    _id: "config",
    "members" : [
    {
        "_id": 0,
        "host" : "mongo01.com:27019"
    },
    {
        "_id": 1,
        "host" : "mongo03.com:27019"
    },
    {
        "_id": 2,
        "host" : "mongo05.com:27019"
    }
    ]
})
```

#### 搭建mongos

- 在mongo01.com / mongo03.com / mongo05.com上执行以下命令：

```apl
# 启动mongos,指定config复制集
mongos --bind_ip 0.0.0.0 --logpath /data/mongos/mongos.log --port 27017 --fork \
--configdb config/mongo01.com:27019,mongo03.com:27019,mongo05.com:27019
```

#### mongos加入第一个分片

```apl
# 连接到mongos
mongo mongo01.com:27017
# 添加分片
mongos>sh.addShard("shard1/mongo01.com:27010,mongo03.com:27010,mongo05.com:27010")
# 查看mongos状态
mongos>sh.status()
```

![](https://note.youdao.com/yws/public/resource/fa7b18a78fa8a4d1961729b043388b57/xmlnote/35B63A223F9545CBBC76ACEC3E4F3F29/35465)

#### 创建分片集合

```apl
连接到mongos, 创建分片集合
mongo mongo01.com:27017
mongos>sh.status()
# 为了使集合支持分片，需要先开启database的分片功能
mongos>sh.enableSharding("company")
# 执行shardCollection命令，对集合执行分片初始化
mongos>sh.shardCollection("company.emp", {_id: 'hashed'})
mongos>sh.status()

# 插入测试数据
 use company
for (var i = 0; i < 10000; i++) {
    db.emp.insert({i: i});
}
# 查询数据分布
db.emp.getShardDistribution()
```

#### 创建第二个分片的复制集

- 在mongo02.com / mongo04.com / mongo06.com上执行以下命令：

```apl
mongod --bind_ip 0.0.0.0 --replSet shard2 --dbpath /data/shard2/db  \
--logpath /data/shard2/log/mongod.log --port 27011 --fork \
--shardsvr --wiredTigerCacheSizeGB 1
```

#### 初始化第二个分片的复制集

```apl
# 进入mongo shell
mongo mongo06.com:27011
# shard2复制集节点初始化
rs.initiate({
    _id: "shard2",
    "members" : [
    {
        "_id": 0,
        "host" : "mongo06.com:27011"
    },
    {
        "_id": 1,
        "host" : "mongo02.com:27011"
    },
    {
        "_id": 2,
        "host" : "mongo04.com:27011"
    }
    ]
})
# 查看复制集状态
rs.status()
```

#### mongos加入第二个分片

```apl
# 连接到mongos
mongo mongo01.com:27017
# 添加分片
mongos>sh.addShard("shard2/mongo02.com:27011,mongo04.com:27011,mongo06.com:27011")
# 查看mongos状态
mongos>sh.status()
```

### 使用分片集群

- 为了使集合支持分片，需要先开启 database 的分片功能

```apl
sh.enableSharding("shop")
```

- 执行 shardCollection 命令，对集合执行分片初始化
  - **shop.product集合将productId作为分片键，并采用了哈希分片策略**
  - numInitialChunks:4，表示将初始化4个chunk
  - **numInitialChunks必须和哈希分片策略配合使用**，而且，这个选项只能用于空集合，如果已经存在数据则会返回错误

```apl
sh.shardCollection("shop.product",{productId:"hashed"},false,{numInitialChunks:4})。
```

#### 向分片集合写入数据

```apl
db=db.getSiblingDB("shop");
var count=0;
for(var i=0;i<1000;i++){
    var p=[];
    for(var j=0;j<100;j++){
        p.push({
            "productId":"P-"+i+"-"+j,
            name:"羊毛衫",
            tags:[
                {tagKey:"size",tagValue:["L","XL","XXL"]},
                {tagKey:"color",tagValue:["蓝色","杏色"]},
                {tagKey:"style",tagValue:"韩风"}
            ]
        });
    }
    count+=p.length;
    db.product.insertMany(p);
    print("insert ",count)
}   
```

#### 查询数据的分布

```apl
db.product.getShardDistribution()
```

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/F8285EA0CF56408AB5EF5EB319CC7B02/42485)

### 分片策略

- 通过分片功能，可以将一个非常大的集合分散到不同的分片上，如图：

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/1131BE2A72B145FD8135A23477D5044F/41977)

- 一个集合在拆分后如何存储、读写，决定于该集合的分片策略设定

#### chunk

- 数据块的意思，一个chunk代表了集合中的“一段数据”，举例下图：

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/BB6097CD828348E39319FB492E5044A6/41988)

- chunk描述的是范围区间，例如，db.users使用了userId作为分片键，那chunk就是userId的各个值（或哈希值）的连续区间。
- **集群在操作分片集合时，会根据分片键找到对应的chunk，并向该chunk所在的分片发起操作请求**
- 而chunk的分布在一定程度上会影响数据的读写路径，这由以下两点决定：
  - chunk的切分方式，决定如何找到数据所在的chunk
  - chunk的分布状态，决定如何找到chunk所在的分片

#### 分片算法

- **chunk切分是根据分片策略进行实施的，分片策略的内容包括分片键和分片算法**
- 当前，MongoDB支持两种分片算法：

##### 范围分片（range sharding）

- 假设集合根据x字段来分片，x的完整取值范围为[minkey, maxKey]（x为正整数，这里是最小值、最大值）
- 其将整个取值范围划分为多个chunk，例如：
  - chunk1 为 [0, 10)
  - chunk2 为 [10, 20]

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/5BE0C4CFE8014111B11E25FE609176F7/42173)

- **范围分片能很好地满足范围查询的需求，缺点在于，如果Shard Key有明显递增（递减）趋势，则新插入的文档会分布到同一个chunk，此时写压力会集中到一个节点，从而导致单点的性能瓶颈**
  - 一些常见的导致递增的key如下：
    - 时间值
    - ObjectId，自动生成的_id由时间、计数器组成
    - UUID，包含系统时间、时钟序列
    - 自增整数序列

##### 哈希分片（hash sharding）

- **哈希分片会事先根据分片键计算出一个新的哈希值（64位整数），再根据哈希值按照范围分片的策略进行chunk的切分**
  - 适用于日志、物联网等高并发场景

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/3B7D5A530DD840508AC02DBBC1B028DC/42170)

- 哈希分片与范围分片是互补的，**由于哈希算法保证了随机性，所以文档可以更加离散的分布到各个chunk上，避免了集中问题**
  - 然而，**在执行一些范围查询时，哈希分片并不是高效的**
- 哈希分片与范围分片的另一个区别是：**哈希分片只允许单个字段、范围分片允许多字段作为分片键**
- 4.4以后的版本，可以将单个字段的哈希分片和一个到多个的范围分片键字段来进行组合，比如指定 x:1，y是哈希的方式

```apl
{ x : "hashed" } 
{x : 1 , y : "hashed"} // 4.4 new
```

#### 分片标签

- **MongoDB允许通过分片添加标签（tag）的方式来控制数据分发**
- 一个标签可以关联到多个分片区间（TagRange）
- 均衡器会优先考虑chunk是否正处于某个分片区间上（被完全包含），如果是则会将chunk迁移到分片区间所关联的分片，否则按一般情况处理
- 分片标签适用于一些特定场景，例如：集群中可能同时存在OLAP和OLTP处理，一些系统日志的重要性相对较低，而主要以少量的统计分析为主。
  - 为了便于扩展，可能希望将日志与实时类的业务数据分开，此时就可以使用标签。
- 为了让分片拥有指定的标签，需执行addShardTag命令

```apl
sh.addShardTag("shard01","oltp")
sh.addShardTag("shard02","oltp")
sh.addShardTag("shard03","olap")
```

- 实时计算的集合应该属于 OLTP 标签，声明 TagRange

```apl
sh.addTagRange("main.devices",{shardKey:MinKey},{shardKey:MaxKey},"oltp")
```

- 而离线计算的集合，则属于 OLAP 标签

```apl
sh.addTagRange("other.systemLogs",{shardKey:MinKey},{shardKey:MaxKey},"olap")
```

- main.devices 集合将被均衡地分布到 shard01、shard02分片上，而 other.systemLogs 集合将被单独分发到 shard03 分片上

#### 分片键（ShardKey）的选择

- 在选择分片键时，需要根据业务的需求及范围分片、哈希分片的不同特点来进行权衡，一般关键因素：
  - **分片键的基数（cardinality），取值基数越大越有利于扩展**
    - 以性别作为分片键，数据最多被拆分为2份
    - 以月份作为分片键，数据最多被拆分为12份
  - 分片键的取值分布应该尽可能均匀
  - **业务读写模式，尽可能分散写压力，而读操作尽可能来自一个或少量的分片**
  - 分片键应该能适应大部分的业务操作

#### 分片键的约束

- ShardKey 必须是一个索引。非空集合须在 ShardCollection 前创建索引；空集合ShardCollection自动创建索引
- 4.4版本前：
  - ShardKey大小不能超过512 Bytes
  - 仅支持单字段的哈希分片键
  - Document中必须包含ShardKey
  - ShardKey包含的Field不可以修改
- 4.4版本之后：
  - ShardKey大小无限制
  - 支持复合哈希分片键
  - Document中可以不包含ShardKey，插入时被当做 Null 处理
  - 为 ShardKey 添加后缀 refineCollectionShardKey 命令，可以修改 ShardKey 包含的 Field
- 在4.2版本之前，ShardKey对应的值不可以修改，4.2之后，如果ShardKey为非_id字段，则可以修改

### 数据均衡

#### 均衡的方式

- 均衡性包括两个方面：
  1. **所有的数据应均匀地分布于不同的chunk上**
  2. **每个分片上的chunk数量尽可能是相近的**
- 第一点由业务场景和分片策略来决定，关于第二点，有以下两种选择：

##### 手动均衡

1. 在初始化集合时预分配一定数量的chunk（仅适用于哈希分片），比如10个分片分配100个chunk，那每个分片拥有10个chunk
2. 可以通过 splitAt、moveChunk 命令进行手动切分、迁移

##### 自动均衡

- 开启MongoDB集群的自动均衡功能。**均衡器会在后台对各分片的chunk进行监控，一旦发现了不均衡状态就会自动进行chunk的搬迁，以达到均衡**
- 其中，chunk不均衡通常来自于两方面的因素：
  - 一方面，在没有人工干预的情况下，chunk会持续增长并发生分裂（split），而不断分裂的结果就会出现数量上的不均衡
  - 另一方面，在动态增加分片服务器时，也会出现不均衡的情况。自动均衡是开箱即用的，可以极大简化集群的管理工作

#### chunk分裂

- 默认情况下， 一个chunk的大小为64MB，该参数由配置的chunksize参数指定。
- 如果持续地向该chunk写入数据，并导致数据量超过了chunk大小，则MongoDB会自动进行分裂，将该chunk切分为两个相同大小的chunk

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/961761E46E054C5F8C89986FD739B344/42051)

- **chunk分裂是基于分片键进行的，如果分片键的基数太小，则可能因为无法分裂而出现jumbo chunk（超大块）的问题**
  - 例如：对 db.users 使用 gender(性别) 作为分片键，同种性别的用户数可能达到数千万，分裂程序并不知道该如何对分片键（gender）的一个单值进行切分，因此最终导致在一个chunk上集中存储了大量的user记录（总大小超过64MB）
- **jumbo chunk对水平扩展有负面作用，该情况不利于数据的均衡，业务上应尽可能避免**，一些写入压力过大的情况可能会导致chunk多次失败（split），最终当chunk中的文档数大于 1.3xavgObjectSize 时，会导致无法迁移。此外在一些老版本中，如果chunk中的文档数超过250000个，也会导致无法迁移

#### 自动均衡

- MongoDB的数据均衡器运行于 Primary Config Server（配置服务器的主节点）上，而该节点也同时会控制chunk数据的搬迁流程

![](https://note.youdao.com/yws/public/resource/16c445f1d16c79e23d1205a79110d3c5/xmlnote/DB91B049D98F40168428B4F8F58E2611/42063)

- 流程说明：
  1. 分片shard0在持续的业务写入压力下，产生了chunk分裂
  2. 分片服务器通知Config Server进行元数据更新
  3. Config Server的自动均衡器对chunk分布进行检查，发现shard0和shard1的chunk数差异达到了阈值，向shard0下发moveChunk命令以执行chunk迁移
  4. shard0执行指令，将指定数据块复制到shard1。该阶段会完成索引、chunk数据的复制，而且在整个过程中业务侧对数据的操作仍然会指向shard0；所以，在第一轮复制完毕之后，目标shard1会向shard0确认是否还存在增量更新的数据，如果存在则继续复制
  5. shard0完成迁移后发送通知，此时Config Server开始更新元数据，将chunk的存量更新为目标shard1。在更新完元数据后，并确保没有关联cursor的情况下，shard0会删除被迁移的chunk副本
  6. Config Server通知 mongos 服务器更新路由表。此时，新的业务请求会被路由到shard1

##### 迁移阈值

- 均衡器对于数据的 "不均衡状态" 判定，是根据两个分片上的chunk个数差异来进行的

| chunk个数 | 迁移阈值 |
| --------- | -------- |
| 少于20    | 2        |
| 20～79    | 4        |
| 80及以上  | 8        |

##### 迁移速度

- 数据均衡的整个过程并不是很快，影响MongoDB均衡速度的几个选项如下：
  - **_secondaryThrottle：用于调整迁移数据写到目标分片的安全级别**。如果没有设定，则会使用w: 2选项，即至少一个备节点确认写入迁移数据后才算成功。从MongoDB3.4版本后，被默认设定为 false，chunk迁移不再等待备节点写入确认
  - _waitForDelete：在chunk迁移完成后，源分片会将不再使用的chunk删除。如果 _waitForDelete 是true，那么均衡器需要等待chunk同步删除后，才进行下一次迁移。**该选项默认为false，这意味着对于旧chunk的清理是异步进行的**
  - 并行迁移数量：在早期版本的实现中，均衡器在同一个时刻只能有一个chunk迁移任务。从MongoDB3.4版本开始，**允许n个分片的集群同时执行n/2个并发任务**
- 随着版本的迭代，MongoDB迁移的能力也在逐步提升。从MongoDB4.0版本开始，支持在迁移数据的过程中并发地读取源端和写入目标端，迁移的整体性能提升了约40%。这样也使得新加入的分片能更快地分担集群的访问读写压力。

#### 数据均衡带来的问题

1. **数据均衡会影响性能**，分片间进行数据块的迁移，很容易带来磁盘I/O使用率飙升，或业务时延陡增等问题。因此，建议尽可能提升磁盘能力，如使用SSD。除此之外，还可以将数据均衡的窗口对齐到业务的低峰期以降低影响。

   - 登陆mongos，在config数据库上更新配置，代码如下：

     - 启用自动均衡器，在每天凌晨2点到4点进行数据均衡操作

   - ```apl
     use config
     sh.setBalancerState(true)
     db.settings.update(
         {_id:"balancer"},
         {$set:{activeWindow:{start:"02:00",stop:"04:00"}}},
         {upsert:true}
     )
     ```

2. **对分片集合中执行count命令可能会产生不准确的结果**，mongos在处理count命令时，会分别向各个分片发送请求，并累加最终结果。如果分片上正在执行数据迁移，则可能导致重复的计算。替代办法是使用 db.collection.countDocuments({}) 方法，该方法会执行聚合操作进行实时扫描，可以避免元数据读取的问题，但需要更长时间。

3. **在执行数据库备份的期间，不能进行数据均衡操作**，否则会产生不一致的备份数据。在备份数据之前，可以通过如下命令确认均衡器的状态：

   1. sh.getBalancerState()：查看均衡器是否开启
   2. sh.isBalancerRunning()：查看均衡器是否正在运行
   3. sh.getBalancerWindow()：查看当前均衡的窗口设定



## MongoDB多文档事务

- **在MongoDB中，对单个文档的操作是原子的**。
- MongoDB支持多文档事务，而使用分布式事务，事务可以跨多个操作、集合、数据库、文档和分片使用
- MongoDB 4.2版本后，全面支持多文档事务，但 **对事务的使用原则应该是：能不用尽量不用**

### 使用事务的原则

- 无论何时，事务的使用总是能避免则避免
- **模型设计先于事务，尽可能用模型设计规避事务**
- 不要使用过大的事务（尽量控制在1000个文档更新以内）
- 当必须使用事务时，尽可能让涉及事务的文档分布在同一个分片上，这将有效提升效率

### MongoDB对事务支持

| 事务属性           | 支持程度                                                     |
| ------------------ | ------------------------------------------------------------ |
| Atomocity 原子性   | 单表单文档 ： 1.x 就支持复制集多表多行：4.0分片集群多表多行：4.2 |
| Consistency 一致性 | writeConcern, readConcern (3.2)                              |
| Isolation 隔离性   | readConcern (3.2)                                            |
| Durability 持久性  | Journal and Replication                                      |

### 使用方法

```apl
try (ClientSession clientSession = client.startSession()) {
   clientSession.startTransaction(); 
   collection.insertOne(clientSession, docOne); 
   collection.insertOne(clientSession, docTwo); 
   clientSession.commitTransaction(); 
 }
```

### writeConcern

- **决定一个写操作落到多少个节点上才算成功**

- 配置语法格式：

  - ```apl
    { w: <value>, j: <boolean>, wtimeout: <number> }
    ```

  - w：数据写入到number个节点才向客户端确认

    - {w:0} 对客户端的写入不需要发送任何确认，适用于性能要求高的场景
    - {w:1} 默认的writeConcern，数据写入到Primary就向客户端发送确认
    - ![](https://note.youdao.com/yws/public/resource/4c81223e709ec11e17d06ef08082cfc0/xmlnote/5120DC7466204ED3A37E6CA49C8A8E0E/44022)
    - {w:"majority"} 数据写入到副本集大多数成员后向客户端发送确认，适用于对数据安全性要求比较高的场景，性能较低
    - ![](https://note.youdao.com/yws/public/resource/4c81223e709ec11e17d06ef08082cfc0/xmlnote/C841CF45D56F45E691F79482D835E049/44020)

  - j：写入操作的 journal 持久化后才向客户端确认

    - 默认值为 {j:false}，如果要求Primary写入持久化了才向客户端确认，则指定该值为 true

  - wtimeout：写入超时时间，仅w的值大于1时有效，写入超时仍未结束，则认为写入失败

#### 测试

- 包含延迟节点的3节点pss复制集

```apl
db.user.insertOne({name:"李四"},{writeConcern:{w:"majority"}})
# 等待延迟节点写入数据后才会响应
db.user.insertOne({name:"王五"},{writeConcern:{w:3}})
# 超时写入失败
db.user.insertOne({name:"小明"},{writeConcern:{w:3,wtimeout:3000}})
```

#### 注意事项

- 虽然多于半数的 writeConcern 都是安全的，但通常只会 **设置majority，因为这都是等待写入延迟时间最短的选择**
- **不要设置writeConcern等于总结点数**，因为一旦有一个节点失败，所有写操作都会失败
- writeConcern 虽然会增加写操作延迟时间，但并不会显著增加集群压力，因此无论是否等待，写操作最终都会复制到所有节点上。设置 writeConcern 只是让写操作等待复制后再返回而已
- **应对重要数据应用 {w: "majority"}，普通数据可以应用 {w:1} 以确保最佳性能**