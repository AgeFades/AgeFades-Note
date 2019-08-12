# 黄康 - ES 快速开发

## 核心概念

### Cluster

```shell
集群由一个或多个节点组成，一个集群有一个默认名称"Elasticsearch",不同集群，集群名称应唯一
```

### Node

```shell
# 节点是集群的一部分
Master 节点 # 存元数据
Data 节点 # 存数据
Ingest 节点 # 可在数据真正 index 前，通过配置 pipline 拦截器对数据 ETL
Coordinate 节点 # 协调节点，如接收搜索请求，并将请求转发到数据节点，每个数据节点在本地执行请求并将结果返回给协调节点，协调节点将每个数据节点的结果汇总并返回给客户端。每个节点默认都是一个协调节点，当将 node.master，node.data，node.ingest 设置为 false 时，该节点仅用作协调节点

# 注意 Coordinate Tribe 是一种特殊类型的协调节点，可连接到多个集群并在所有连接的集群中执行搜索和其他操作。
```

### 分片和副本

```shell
# 副本(replica) 是分片的副本
# 分片有主分片(primary Shard) 和副本分片(replica Shard) 之分
一个 Index 数据在物理上被分布在多个主分片中，每个主分片只存放部分数据
每个主分片可以有多个副本，叫副本分片，是主分片的复制

# 注意
1. 一个 document 只存在于某个 primary shard 以及其对应的 replica shard 上，不会在多个 primary shard 上

2. 默认一个 index 有5个主分片，每个主分片都有一个副本，整个 index 就有10个分片，要保证整个集群健康，就需要至少 2 个节点，因为主分片和副本不能在同一台机器上

3. 主分片的数量在创建索引后不能再被修改，副本分片的数量可以改变

4. 每个分片 shard 都是一个完整的 lucene 实例，有完整创建索引和处理请求的能力

5. 分片有助于 es 水平扩展，副本一般用来容灾，相当于主分片的 HA

6. 分片和副本还有助于提高并行搜索的性能，因为可以在所有副本上并行执行搜索
```

### Index

```shell
一个 Index[索引] 可以理解成 MySQL 中的 Database
```

### Type

```shell
一种 Type[类型] 可以理解成 MySQL 中的 Table
# 注意
	ES 5.x 中一个 Index 可以有多种 Type
	ES 6.x 中一个 Index 只能有一种 Type
	ES 7.x 以后，将移除 Type 这个概念
```

### Document

```shell
一个 Document[文档] 可以理解成 MySQL 中的 Row
```

### Filed

```shell
一个 Field[字段] 可以理解成 MySQL 中的 Column
```

### Mapping

```shell
Mapping[映射] 可以理解成 MySQL 中的表结构
```

### Query DSL

```shell
可以理解成 MySQL 中的 SQL 语句用来编写查询数据的语句以及查询条件和处理等
```

### Segment

```shell
一个 shard 包含多个 segment，每个 segment 都是倒排索引
查询时，每个 shard 会把所有 segment 结果汇总作为 shard 的结果返回

# 注意
1. 写入 ES 时，一方面会把数据写入到内存 Buffer 缓冲，为防止 Buffer 中数据丢失，另一方面会同时把数据写入到 translog 日志文件

2. 每隔 1s,数据从 Buffer 被写入到 segment file，直接写入 os chache

3. os cache segment file 被打开供 search 使用，然后内存 Buffer 被清空

4. 随着时间推移，os cache 中 segment file 越来越多，translog 日志文件也越来越大，当 translog 日志文件大到一定程度的时候就会触发 flush 操作

5. flush 操作将 os cache 中 segment file 持久化到磁盘并删除 translog 日志文件

6. 当 segment 增多到一定程度，会触发 ES 合并segment，将许多小的 segment 合并成大 segment，提高性能
```

## Restful Api

### 新建 Mapping

```shell
# 新建 Mapping 3个分片，0个副本，映射类型是 books
PUT /myindex
{
 "settings":{
	"number_of_shards" : 3,
	"number_of_replicas" : 0
},
 "mappings":{
  "books":{
    "properties":{
        "title":{"type":"text"}, # 字段名和字段类型
        "name":{"type":"text","index":false},
        "publish_date":{"type":"date","index":false},
        "price":{"type":"double"},
        "number":{"type":"integer"}
    }
  }
 }
}
```

### 查看 Mapping

```shell
GET /myindex
```

### 新建 Mapping 属性

#### 定制类型

定制 dynamic，定制自己的字段，该 Mapping 类型为 Integer，它的 dynamic 为 true

```shell
"number":{
    "type":"integer",
    "dynamic":true
}

# 注释
"dynamic":true # 遇到陌生字段，就进行 dynamic mapping
"dynamic":false # 遇到陌生字段，就忽略
"dynamic":strict # 遇到陌生字段，就报错

```

