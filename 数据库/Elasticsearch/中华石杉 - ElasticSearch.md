# 中华石杉 - ElasticSearch

## 搭建

```shell
# 拉取镜像，内含 ES 以及 Kibana，当前最新版本均为 7.1，此版本已经取消 type 概念
docker pull nshou/elasticsearch-kibana 

# 创建容器
docker run -d -p 9200:9200 -p 9300:9300 -p 5601:5601 --name eskibana nshou/elasticsearch-kibana

# 浏览器访问 localhost 5601，找到 Dev Tools 进行 Restful 与 ES 交互
```

## 简单 CRUD

```shell
# 检查集群状况
GET _cat/health
```

![UTOOLS1573786496348.png](https://i.loli.net/2019/11/15/G5wd3lmLBCaZ4tJ.png)

```shell
# 集群 Status
green: 每个索引的 primary shard 和 replica shard 都是 active 状态的

yellow: 每个索引的 primary shard 都是 active 状态的，但是部分 replica shard 不是 active 状态，处于不可用的状态

red: 不是所有索引的 primary shard 都是 active 状态的，部分索引有数据丢失了。
```

```shell
# 快速查看集群中有哪些索引
GET _cat/indices
```

```shell
# 创建索引
PUT /index
```

![UTOOLS1573786933156.png](https://i.loli.net/2019/11/15/z3pt5S8cswGWYIg.png)

```shell
# 删除索引
DELETE /index
```

```shell
# 创建文档
PUT /index/_doc/id
{
	JSON 数据
}
```

![UTOOLS1573787334261.png](https://i.loli.net/2019/11/15/SG6pnvOgeKArzfx.png)

```shell
# 更新文档 <全量替换>
PUT /index/_doc/id
{
	JSON 数据
}
```

![UTOOLS1573787463171.png](https://i.loli.net/2019/11/15/roQvH9MkZIw7Phi.png)

```shell
# 查看文档
GET /index/_doc/id
```

```shell
# 删除文档
DELETE /index/_doc/id
```

## 多种搜索方式

```shell
# query string search <生产环境不适用>
	# search 参数都是以 http 请求的 query string 附带来的

took: 耗费了几毫秒
time_out: 是否超市
_shards: 搜索请求分配到了多少个 primary shard <或者是它的某个 replica shard>
hits.total: 查询结果的数量
hits.max_score: score 的含义，就是 document 对于一个 search 的相关度的匹配分数，越相关，就越匹配，分数也高
hits.hits: 包含了匹配搜索的 document 的详细数据
```

![UTOOLS1573798223522.png](https://i.loli.net/2019/11/15/pA7so8IJqrzCh9G.png)	

```shell
# query DSL
	# 将所有的请求条件都放入请求体中，用 json 格式构建查询语法
	
# ES 5.X 版本后，对排序和聚合等操作，用单独的数据结构(fielddata) 缓存到内存中，默认不开启，需要单独开启

query: 查询条件
	match_all: 匹配所有
	match: 匹配

sort: 排序条件

from: 分页条件
	from: 从第几个开始查
	size: 查多少条
	
_source: 指定查询的字段
	_source: ["xx","xx"]

highlight: 高亮
```

![UTOOLS1573798981707.png](https://i.loli.net/2019/11/15/7CiUutRNHoSIrZj.png)

![UTOOLS1573799446719.png](https://i.loli.net/2019/11/15/skQyHcS4d7Y5n1T.png)

```shell
# query filter
	# 搜索过滤
query: 查询条件
	bool: 布尔类型匹配
		must:	必须的
			match: 匹配条件
		filter:
			range: 范围条件
```

```shell
# query filter 示例:
GET /swordsman/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "name": "swordsman"
        }
      },
      "filter": {
        "range": {
          "age": {
            "gte": 20
          }
        }
      }
    }
  }
}
```

```shell
# full-text search <全文检索,分词、倒排索引、相关度算法分析>
	# 将搜索条件拆解开来，去倒排索引里一一匹配，只要匹配上任意一个拆解后的单词，就可以作为结果返回
	# 举例: name 字段被拆解成多个单词，建立倒排索引
GET /swordsman/_search
{
  "query": {
    "match": {
      "name": "swordsman Hi"
    }
  }
}
```

```shell
# phrase search <短语搜索>
	# 跟全文检索相反，要求在指定的字段文本中，必须包含一模一样的搜索条件，才可以算匹配，作为结果返回	
	# 举例:
GET /swordsman/_search
{
  "query": {
    "match_phrase": {
      "name": "sowrdsman Hi"
    }
  }
}
```

```shell
# 高亮
	# 指定返回字段高亮显示
	# 举例:
GET /swordsman/_search
{
  "query": {
    "match": {
      "name": "swordsman Hi"
    }
  },
  "highlight": {
    "fields": {
      "name":{}
    }
  }
}
```

## 聚合分析

```shell
size: 0 表示不需要返回参与聚合的数据
aggs: 聚合
	name: 任意起的聚合名字
		agg_type: 聚合的类型

# 举例，查出每个年龄的人数
GET /swordsman/_search
{
 "size": 0,
 "aggs": {
   "GROUP_BY_AGE": {
     "terms": {
       "field": "age"
     }
   }
 }
}

# 举例，查出每个年龄下名字中包含 hi 的数据
GET /swordsman/_search
{
 "size": 10,
 "query": {
   "match": {
     "name": "hi"
   }
 }, 
 "aggs": {
   "GROUP_BY_AGE": {
     "terms": {
       "field": "age"
     }
   }
 }
}

# 举例，查出每个年龄下的分数平均值，并进行降序排序
GET /swordsman/_search
{
 "size": 0,
 "aggs": {
   "GROUP_BY_AGE": {
     "terms": {
       "field": "age",
       "order": {
         "AVG_SCORE": "desc"
       }
     },
     "aggs": {
       "AVG_SCORE": {
         "avg": {
           "field": "score"
         }
       }
     }
   }
 }
}

# 举例，按照指定的年龄范围区间进行分组，然后在每组内再按照班级进行分组，最后再计算每组的平均成绩
GET /swordsman/_search
{
 "size": 0,
 "aggs": {
   "GROUP_BY_AGE": {
     "range": {
       "field": "age",
       "ranges": [
         {
           "from": 0,
           "to": 20
         },
         {
           "from": 20,
           "to": 40
         }
       ]
     },
     "aggs": {
       "GROUP_BY_CLASS":{
         "terms": {
           "field": "class"
         },
         "aggs": {
           "AVG_SCORE": {
             "avg": {
               "field": "score"
             }
           }
         }
       }
     }
   }
 }
}
```

## ElasticSearch 的基础分布式架构

### ElasticSearch 对复杂分布式机制的透明隐藏特性

```shell
# ElasticSearch 是一套分布式的系统，分布式是为了应对大数据量

# ElasticSearch 隐藏了复杂的分布式机制实现
	# 分片机制: 将 document 插入 es 集群中时，帮我们实现了数据的分片、分片的规则路由
	
	# cluster discover: 集群发现机制，新增节点自动发现集群并加入，并且还接收了部分数据
	
	# shard 负载均衡: 假设3个节点，总共25 个shard 要分配到3个节点上去，es 会自动进行均匀分配，以保持每个节点的均衡的读写负载请求
	
	# shard 副本、请求路由、集群路由、shard 重新分配...
```

### ElasticSearch 的垂直扩容与水平扩容

```shell
# 假设现在6台服务器，每台容纳 1T 数据，但数据量马上要增长到 8T，此时两个方案:
	# 垂直扩容:
		# 加硬盘，将其中两台服务器增加至 2T 硬盘
		
	# 水平扩容:
		# 加服务器，再来两台 1T 硬盘的服务器
		
# 业界经常采用的方案是水平扩容，没有性能瓶颈、投入资金相较于垂直扩容更少
```

### 增加或减少节点时的数据 rebalance

```shell
# 比如 6 台服务器，但是有7个 shard，某台服务器就会存放两个 shard，负载更重，承载的数据和请求量也会更大。此时增加一台服务器节点，ES 会自动调整资源，将多的 shard 分配至新的服务器节点上。

# 减少节点同理
```

### master 节点

```shell
# 管理 es 集群的元数据: 比如说索引的创建和删除、维护索引元数据、节点的增加和移除、维护集群的元数据

# 默认情况下，会自动选择出一台节点，作为 master 节点

# master 节点不会承载所有的请求，所以不会是一个单点瓶颈
```

### 节点平等的分布式架构

```shell
# 节点对等，每个节点都能接收所有的请求

# 自动请求路由

# 响应收集
```

## shard & replica 机制再次梳理

### shard & replica 机制再次梳理

```shell
# index 包含多个shard

# 每个shard 都是一个最小工作单元，承载部分数据，lucene 实例，完整的建立索引和处理请求的能力

# 增减节点时，shard 会自动在 nodes 中负载均衡

# primary shard 和 replica shard，每个 document 肯定只存在于某一个 primary shard 以及其对应的 replica shard 中，不可能存在于多个 primary shard

# replica shard 是 primary shard 的副本，负责容错，以及承担读请求负载

# primary shard 的数量在创建索引的时候就固定了， replica shard 的数量可以随时修改

# primary shard 的默认数量是5，replica 默认是 primary shard * 1，默认有10 个shard，5个 primary shard，5个 replica shard

# primary shard 不能和自己的 replica shard 放在同一个节点上（否则节点宕机，primary shard 和 replica shard 和副本都会丢失，起不到容错的作用），但是可以和其他 primary shard 的 replica shard 放在同一个节点上
```

### 单 node 环境下创建 index 是什么样子的

```shell
# 单 node 环境下，创建一个 index，有3个 primary shard，3个 replica shard
	# 集群 status 是 yellow
	# 此时，只会将 3 个primary shard 分配到仅有的一个 node 上去，另外 3 个replica shard 是无法分配的。
	# 集群可以正常工作，但是一旦出现节点宕机，数据全部丢失，而且集群不可用，无法承接任何请求
	
PUT /Swordsman
{
	"settings": {
		"number_of_shards": 3
		"number_of_replicas": 1
	}
}
```

## 横向扩容过程、如何超出扩容极限、以及如何提升容错性

```shell
# primary & replica 自动负载均衡: 比如说本来 6个shard，2台服务器，一台三个 primary shard，一台 三个 replica shard，此时再增加一个服务器，就会变成每个服务器两个 shard

	# 扩容之后，每个节点的 shard 数量更少，就意味着每个 shard 可以占用节点上更多的资源，IO/CPU/Memory，整个系统，性能会更好
	
# 扩容极限: 6个 shard，最多扩容到6台服务器，每个 shard 都占用单体服务器的所有资源

# 超出扩容极限，动态修改 replica shard 数量，9个 shard<3个 primary，6个 replica>，扩容到9台机器，比3台机器时，拥有3倍的读吞吐量

# 3台机器下，6个 shard 情况下，可以容忍 1个节点宕机，9个 shard 情况下，可以容忍 2个节点宕机，而保证数据不丢失
```

## ElasticSearch 容错机制: master 选举、replica 容错、数据恢复

```shell
# 假设现在3 个Node，9 个Primary shard

# 1. 第一个 Node 宕机，此时缺少1个 primary shard 和2个 replica shard，集群状态变为 red

# 2. 容错第一步:
	# 剩下的两个 node 进行 master 选举，补充新的 master 节点承担起 master 责任
	
# 3. 容错第二步
	# 剩下两个活着的节点，会将丢失掉的 primary shard 的某个 replica shard 提升为 primary shard，此时集群状态变为 yello，但是少了 replica shard
	
# 4. 容错第三部
	# 重启宕机节点，master 会将缺失的副本都 copy 一份到该 node 上，并且同步数据，此时集群状态变为 green。
```

## 初步解析 document 的核心元数据

```shell
# _index
	# 代表一个 document 存放在哪个 index 中
	# 类似的数据放在一个索引中，非类似的数据放不同索引
	# index 中包含了很多类似的 document
	# 索引名称必须是小写的，不能用下划线开头，不能包含逗号
	
# _type
	# ES 5.x 以后已经取消了这个概念
	# PUT 创建数据的结果所有 _type 都统一为了 _doc
	
# _id
	# 代表 document 的唯一标识，与 index 一起，定位唯一的 document
	# 可以手动指定 document 的 id，也可以不指定，由 es 自动为我们创建一个 id
```

## document id 的手动指定与自动生成两种方式解析

```shell
# 手动指定 document id
	# 一般来说，是从其他系统中，导入数据到 es 中，会采取这种方式
PUT /index/_doc/id

# 自动生成 document id
	# 自动生成的id,长度为20个字符，url 安全，base64 编码，GUID，分布式系统并行生成时不可能会发生冲突
POST /index/_doc
```

## document 的 _source 元数据以及定制返回结果解析

```shell
# GET /index/_doc/id 时，结果的 _source 即创建 document 时的 request body 中的 JSON 对象数据

# 如果想指定返回哪些 field:
	# GET /index/_doc/id?_source=xxx,xxx
```

## document 的全量替换、文档删除等操作的分析

```shell
# document 的全量替换
	# 语法与创建文档是一样的
	# document 是不可变的，如果要修改 document 的内容，第一种方式就是全量替换，直接对 document 重新建立索引，替换里面所有的内容
	# es 会将老的 document 标记为 deleted，然后新增我们给定的一个 document，当我们创建越来越多的 document 时，es 会在适当的时机在后台自动删除标记为 deleted 的document 
```

```shell
# document 的删除操作
	# 将 document 的状态标记为 deleted，查询不到
	# 数据量越来越多时，es 会自动将标记为 deleted 的 document 删除
```

## 深度图解 ES 并发冲突问题

![UTOOLS1574055262908.png](https://i.loli.net/2019/11/18/eFXMWzQAJ79bOld.png)

## 深度图解悲观锁与乐观锁两种并发控制方案

![UTOOLS1574062413339.png](https://i.loli.net/2019/11/18/K5RygAcid4ZrG6m.png)

## 图解 ElasticSearch 内部如何基于 _version 进行乐观锁并发控制

![UTOOLS1574063239767.png](https://i.loli.net/2019/11/18/4mQor9lf6nvRTE8.png)

## 基于 external version 进行乐观锁并发控制

```shell
# external version
	# 使用 es 时，可以不用它提供的内部 _version 来进行并发控制，自己实现
```

## 图解 partial update 实现原理

![UTOOLS1574065210323.png](https://i.loli.net/2019/11/18/PwNvtlgsk6TG3VM.png)

## 图解 partial update 乐观锁并发控制原理

![UTOOLS1574065353059.png](https://i.loli.net/2019/11/18/xsUenoP3NA1gDV4.png)

## document 数据路由原理

```shell
# document 路由到 shard 上是什么意思？

# 路由算法: shard = hash(routing) % number_of_primary_shards
	# 举个例子，一个 index 有3个 primary shard，每次 CRUD  一个 document 时，都会带来一个 routing number，默认就是这个 document 的 _id (可能是手动指定，也可能是自动生成)
	
	# 无论 hash 值是几，无论是什么数字，对 number_of_primary_shards 求余数，结果一定在 0 - number_of_primary_shards - 1这个范围内，这就是 primary shard 数量不可变的谜底
```

## document 增删改内部原理

```shell
# 1. 客户端选择一个 node 发送请求过去，这个 node 就是协调节点

# 2. 协调节点对 document 进行路由，将请求转发给对应的 primary shard

# 3. primary shard 处理请求，然后将数据同步到 replica shard

# 4. primary shard 和 replica shard 全部搞定后，协调节点响应结果给客户端
```

## document 查询内部原理

```shell
# 1. 客户端发送请求到任意一个 node，成为协调节点

# 2. 协调进行对 document 进行路由，将请求转发到对应的 node，此时会使用 round-robin 随机轮询算法，在 primary shard 以及所有 replica shard 中随机选择一个，让读请求负载均衡

# 3. 接收请求的 node 返回 document 给 协调节点，响应客户端
```

