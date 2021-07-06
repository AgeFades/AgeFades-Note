# 黑马 - Elastic Stack进阶

## 全文搜索

### 倒排索引

```shell
# 倒排索引源于实际应用中需要根据属性的值来查找记录。

# 索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址。

# 由于不是由记录来确定属性值，而是由属性值来确定记录的位置，因而被称为倒排索引<inverted index>

# 带着倒排索引的文件称为 倒排文件<inverted file>
```

![UTOOLS1567759203112.png](https://i.loli.net/2019/09/06/v9cUgIyQWnOFflq.png)

![UTOOLS1567759222971.png](https://i.loli.net/2019/09/06/QFDna7AIx4GXbWr.png)

### 全文搜索

```shell
# 全文搜索两个最重要的方面是
	# 相关性(Relevance)
		# 评价查询和其结果间的相关程度，并根据这种相关程度对结果排名的一种能力
		# 该计算方式可以使 TF|DF 方法、地理位置邻近、模糊相似、或其他的某些算法
	# 分析(Analysis)
		# 将文本块转换为有区别的、规范化的 token 的一个过程
		# 目的是为了创建倒排索引以及查询倒排索引
```

### 单词搜索

```shell
# 过程说明
	# 检查字段类型
		# 一个 text 类型(指定 IK 分词器)，意味着查询字符串本身也应该被分词
	# 分析查询字符串
		# 将查询的字符串传入 IK 中，match 查询执行的是单个底层 term 查询
	# 查找匹配文档
		# 用 term 查询在倒排索引中查找包含该字符串的文档
	# 为每个文档评分
		# 用 term 查询计算每个文档相关度评分 _score
		# 将 词频 + 反向文档频率 + 字符的长度 相结合的计算方式
```

### 多词搜索

```shell
# 多词默认是 or 的关系，可以显示指定词之间的逻辑关系
{
    "query":"音乐 篮球",
    "operator": "and"
}

# 实际中，更多是只需要符合一定的相似度就可以查询到数据，ES 中 minimum_should_match 指定匹配度
{
    "query":"音乐 篮球",
    "minimum_should_match":"80%"
}
```

### 组合搜索

```shell
# 使用过滤器中讲过的 bool 组合查询

# 评分的计算规则
	# bool 查询会为每个文档计算相关度评分 _score，再将所有匹配的 must 和 should 语句
	# 的分数 _score 求和，最后除以 must 和 should 语句的总数
	# must_not 语句不影响评分，它的作用只是将不相关的文档排除
	
# 默认情况下，should 中的内容不是必须匹配的，如果查询语句中没有 must，那么至少匹配一个
	# 也可以通过 minimum_should_match 参数进行控制
	
POST http://['自己的ip 加 port']/beluga/article/_search
{
    "query":{
        "bool":{
            "should":[
                {
                    "match":{
                        "hobby":"游泳"
                    }
                },
                {
                    "match":{
                        "hobby":"篮球"
                    }
                },
                {
                    "match":{
                        "hobby":"音乐"
                    }
                },
            ],
            "minimum_should_match": 2 # should 中的3个词，至少满足两个
        }
    }
}
```

### 权重

```shell
# 对某些词增加权重来影响该条数据的得分
{
    "query":"音乐",
    "boost":10 # 权重为 10
}
```

## 短语匹配

```shell
# ES 中，短语匹配意味着不仅仅是词要匹配，并且词的顺序也要一致
POST http://['自己的ip 加 port']/beluga/article/_search
{
    "query":{
        "match_phrase":{
            "hobby":{
                "query":"羽毛球篮球"
            }
        }
    }
}

# 如果觉得这样太过苛刻，可以增加 slop 参数，允许跳过 N 个词进行匹配
```

## ES 集群

### 集群节点

```shell
# ES 的集群是由多个节点组成的， 通过 cluster.name 设置集群名称,并且用于区分其他的集群
# 每个节点通过 node.name 指定节点的名称

# 在 ES 中，节点的类型主要有 4 种
	# Master 节点
		# 配置文件中 node.master 属性为true(默认为 true)，就有资格被选为 master 节点
		# master 节点用于控制整个集群的操作。比如创建或删除索引，管理其他非 master 节点
	# data 节点
		# 配置文件中 node.data 属性为 true(默认为 true)，就有资格被设置成 data 节点
		# data 节点主要用于执行数据相关的操作，比如文档的 CRUD
	# 客户端节点
		# 配置文件中 node.master 属性和 node.data 属性均为 false
		# 该节点不能作为 master 节点，也不能作为 data 节点
		# 可以作为客户端节点，用于响应用户请求，把请求转发到其他节点
	# 部落节点
		# 当一个节点配置 tribe.* 的时候，它是一个特殊的客户端
		# 可以连接多个集群，在所有连接的集群上执行搜索和其他操作
```

### 使用 dokcer 搭建集群

```shell
# 创建目录
mkdir /docker/es-cluster && cd /docker/es-cluster
mkdir node01 && mkdir node02
chmod 777 node01 && chmod 777 node02

# 复制安装目录的 elasticsearch.yml 、jvm.options 文件，做如下修改
# node01 配置
cluster.name: es-beluga-cluster
node.name: node01
node.master: true
node.data: true
network.host: ['你自己的ip']
http.port: 9200
discovery.zen.ping.unicast.hosts: ["你自己的ip"]
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"

# node02 配置
cluster.name: es-beluga-cluster
node.name: node02
node.master: false
node.data: true
network.host: ['你自己的ip']
http.port: 9201
discovery.zen.ping.unicast.hosts: ['你自己的ip']
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"

# 启动容器
docker run --name es-node01 \
--net host \
-v /docker/es-cluster/node01/elasticsearch.yml: \
/usr/share/elasticsearch/config/elasticsearch.yml
-v /docker/es-cluster/node01/jvm.options: \
/usr/share/elasticsearch/config/jvm.options
-v /docker/es-cluster/node01/data: \
/usr/share/elasticsearch/data \
elasticsearch:6.7

docker run --name es-node02 \
--net host \
-v /docker/es-cluster/node02/elasticsearch.yml: \
/usr/share/elasticsearch/config/elasticsearch.yml
-v /docker/es-cluster/node02/jvm.options: \
/usr/share/elasticsearch/config/jvm.options
-v /docker/es-cluster/node02/data: \
/usr/share/elasticsearch/data \
elasticsearch:6.7

# 查看集群响应
http://['你自己的ip + port']/_cluster/health

# 集群状态的三种颜色
green # 所有主要分片和复制分片都可用
yello # 所有主要分片可用，但不是所有复制都可用
red   # 不是所有的主要分片都可用
```

### 分片和副本

```shell
# 为了将数据添加到 ES,我们需要索引<存储关联数据的地方>。

# 实际上，索引只是一个用来指向一个或多个分片<shards>的逻辑命名空间

	# 一个分片是一个最小级别工作单元，它只是保存了索引中所有数据的一部分。
	# 一个分片就是一个 Lucene 实例，并且它本身就是一个完整的搜索引擎，App 不会和它直接通信
	# 分片可以是主分片或者是复制分片
	# 索引中的每个文档属于一个单独的主分片，所以主分片的数量决定了索引最多能存储多少数据
	# 复制分片只是主分片的一个副本，它可以防止硬件故障导致的数据丢失，同时可以提供读请求
	# 当索引创建完成的时候，主分片的数量就固定了，但是复制分片的数量可以随时调整
```

### 故障转移

```shell
# 集群节点需为奇数<3个以上>，主节点宕机时，集群自动选举出下一个 Master 节点提供服务
```

### 脑裂问题

![UTOOLS1567994449529.png](https://i.loli.net/2019/09/09/xBaQNMe2TGRw8Ku.png)

```shell
# 解决思路
不能让节点很容易的变成 master，必须有多个节点认可后才可以

设置 minmum_master_nodes 的大小为2
	# 官方推荐 : (N/2)+1,N 为集群中节点数
```

### 分布式文档

#### 路由

```shell
# 问题 : 当集群保存文档时，文档该存储到哪个节点？随机？轮询？

# 实际上，在 ES 中，会采用计算的方式来确定存储到哪个节点，计算公式如下
shard = hash(routing) % number_of_primary_shards
	# routing 是一个任意字符串，默认是 _id 也可以自定义
	# routing 哈希后生成数字，再 % 主切面的数量得到一个余数,
	# 该数范围永远是 0 - 主分片数-1，即该文档所在的分片
	# 这就是为什么创建了主分片后，不能修改的原因
```

#### 文档的写操作

```shell
# 新建、索引、删除请求都是写操作，必须在主分片上成功完成才能复制到相关的复制分片上。
	# 1. 客户端给 Node1 发送写操作请求
	# 2. 节点使用文档的_id 确定文档属于分片0，它转发请求到 Node3,分片 0 位于该节点上
	# 3. Node3 在主分片上执行请求，如果成功，它转发请求到 Node1、Node2 的复制节点上
		# 当所有的复制节点报告成功，Node3 报告成功给请求节点，请求节点再报告给客户端
```

#### 搜索文档<单个文档>

```shell
# 文档能够从主分片或任意一个复制分片被检索
	# 1. 客户端给 Node1 发送 get 请求
	# 2. 节点使用文档的 _id 确定文档属于分片0，分片对应的复制分片在三个节点上都有
		# 此时，转发请求到 Node2
	# Node2 返回 Document 给 Node1 然后返回给客户端

# 对于读请求，为了平衡负载，请求节点会为每个请求选择不同的分片，它会循环所有分片副本
```

#### 全文搜索

```shell
# 分为两个阶段 搜索<query> + 取回<fetch>
	# 搜索
		# 1. 客户端发送一个 search 请求给 Node3,Node3 创建了一个长度为 from+size 的空优秀级队
		# 2. Node3 转发这个搜索请求到索引中每个分片的原本或副本，每个分片在本地执行这个查询并且将
			# 结果发送到一个大小为 from+size 的有序本地优先队列里去
		# 3. 每个分片返回 document 的 id 和它优先队列里的所有 document 的排序值给协调节点Node3
			# Node3把这些值合并到自己的优先队列里产生全局排序结果
	# 取回
		# 1. 协调节点分辨出哪个 document 需要取回，并且向相关分片发出 Get 请求
		# 2. 每个分片加载 document 并且根据需要丰富它们，然后再将 document 返回协调节点。
		# 3. 一旦所有的 document 都被取回，协调节点会将结果返回给客户端
```

## Beats

```shell
# 官网 https://www.elastic.co/cn/products/beats
```

