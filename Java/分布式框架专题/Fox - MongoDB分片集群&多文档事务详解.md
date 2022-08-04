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