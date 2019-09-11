# 黑马 - Elastic Stack入门

## 简介

```shell
# ELK -> Elasticsearch | Logstash | Kibana 组成

# 发展中，加入新成员 Beats，不能叫 ELKB<不然以后一直这样加下去吗？>

# 所以统称为 Elastic Stack <Elastic 技术栈>
```

## 组成

![UTOOLS1567660924996.png](https://i.loli.net/2019/09/05/gEjqaYmV4FOtJnK.png)

## 组件介绍

### Elasticsearch

```shell
# 基于Java

# 特点
开源 | 分布式 | 搜索引擎 | 零配置 | 自动发现 | 索引自动分片 |
索引副本机制 | restful | 多数据源 | 自动搜索负载 等
```

### Logstash

```shell
# 基于 Java

# 特点
开源 | 收集 | 分析 | 存储
```

### Kibana

```shell
# 基于 Node.js

# 特点
开源 | 免费 | 友好的日志分析界面 | 汇总 | 分析 | 搜索
```

### Beats

```shell
# elastic 公司开源的一款采集系统监控数据的代理 agent

# 是在被监控服务器上，以客户端形式运行的数据收集器的统称

# 可以直接将数据发送给 Elasticsearch 或通过 Logstash 发送给 Elasticsearch

# Beats 由如下部分组成
Packetbeat # 是一个网络数据包分析器，用于监控、收集网络流量信息
		   # 嗅探服务器之间的流量，解析应用层协议，并关联到消息的处理，支持 ICMP、DNS、HTTP、MySQL
		   # PostgreSQL、Redis、MongoDB、Memcache 等协议
		   
Filebeat   # 用于监控、收集服务器日志文件，其已取代 logstash forwarder

Metricbeat # 可定期获取外部系统的监控指标信息，其可以监控、收集Apache、HAProxy、MongoDB、MySQL
		   # Nginx、PostgreSQL、Redis、System、Zookeeper 等服务
		   
Winlogbeat # 用于监控、收集 Windows 系统的日志信息
```

## Elasticsearch

```shell
# 基于 Lucene 的搜索服务器<倒排索引>

# 官网
https://www.elastic.co/cn/products/elasticsearch
```

### 基本概念

#### 索引(index)

```shell
# 索引是 ES 对逻辑数据的逻辑存储，所以可以被分为更小的部分

# 可以将索引看成 MySQL 的 Table，索引的结构是为快速有效的全文索引准备的，特别是它不存储原始值

# 可以将索引存放在一台机器，或分散在多台机器上

# 每个索引有一或多个分片(shard)，每个分片可以有多个副本(replica)

# 6.5 以后已经取消了 Type 的概念
```

#### 文档(document)

```shell
# 存储在 ES 中的主要实体叫文档，可以看成 MySQL 的一条记录

# ES 与 Mongo 的 document 类似，都可以有不同的结构，但 ES 相同字段必须有相同类型

# document 由多个字段组成，每个字段可能多次出现在一个文档里，这样的字段叫多值字段(multivalued)

# 每个字段的类型，可以使文本、数值、日期等。

# 字段类型也可以是复杂类型，一个字段包含其他子文档或者数组

# 在 ES 中，一个索引对象可以存储很多不同用途的 document，例如一个博客App中，可以保存文章和评论

# 每个 document 可以有不同的结构

# 不同的 document 不能为相同的属性设置不同的类型，例 : title 在同一索引中所有 Document 都应该相同数据类型
```

#### 映射(mapping)

```shell
# 所有文档写进索引之前，都会先进行分析，如何将输入的文本分割为词条、哪些词条又会被过滤、这种行为叫映射

# 一般由用户自己定义规则
```

### Restful API

#### 创建非结构化索引

```shell
# Lucene 中，创建索引是需要定义字段名称以及字段类型的

# ES 中提供非结构化索引，实际上在底层 ES 会进行结构化操作，对用户透明

PUT http://['你自己的Ip加Port']/test
{
    "settings":{
        "index":{
            "number_of_shards":"2", # 分片数
            "number_of_replicas":"0" # 副本数
        }
    }
}
```

#### 删除索引

```shell
DELETE http://['你自己的Ip加Port']/test
{
    "acknowledged": true
}
```

#### 插入数据

```shell
POST http://['你自己的Ip加Port']/{索引}/{类型}{id}

POST http://['你自己的Ip加Port']/beluga/article/1168715220370198528
{
        "id": "1168715220370198528",
        "createTime": "2019-09-03 10:39:56",
        "updateTime": "2019-09-05 14:20:59",
        "title": "快乐的一只小青蛙",
        "author": "AgeFades",
        "updateAuthor": "AgeFades",
        "description": "Test Article",
        "coverUrl": "http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIfHxYV8zPX8nicichsT5XsDRGFmcrqVApicEUZDlIp1EIoFRFt84JibtTXArkupK7ibO7MQjZU7RVCNrw/132",
        "body": "<p>天行健，君子以自强不息1</p>",
        "status": 1
}

# 路径后不跟 Id ,ES 自动生成Id
```

#### 更新数据

```shell
# ES 中，文档数据是不为修改的，但是可以覆盖
PUT http://['自己的ip 加 port']/beluga/article/1168715220370198528
{
	"id":"123"
}

# 局部更新
# 1. 从旧文档中检索 JSON 2. 修改它 3. 删除旧文档 4. 索引新文档
POST http://['自己的ip 加 port']/beluga/article/1168715220370198528/_update
{
	"doc":{
        "id":"123"
	}
}
```

#### 删除数据

```shell
DELETE http://['自己的ip 加 port']/beluga/article/1168715220370198528

# 删除一个文档并不会立即从磁盘上移除，它只是被标记成已删除

# ES 将会在你之后添加更多索引的时候才会在后台进行删除内容的清理
```

#### 搜索数据

```shell
GET http://['自己的ip 加 port']/beluga/article/1168715220370198528

# 搜索全部数据,默认返回 10 条数据
GET http://['自己的ip 加 port']/beluga/article/_saerch

# 关键字搜索数据
GET http://['自己的ip 加 port']/beluga/article/_saerch?q=id:123

# DSL<Domain Specific Language 特定领域语言> 搜索，JSON 请求体形式

# 属性匹配查询
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "match":{
            "id": 123
        }
    }
}

# 范围查询
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "bool": {
            "filter": {
                "range":{
                	"age":{
                        "gt": 30 # 查询年龄大于30岁
                	}
                }
            },
            "must":{
                "match":{
                    "sex":"男" # 性别为男
                }
            }
        }
    }
}

# 全文搜索
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "match":{
            "name": "张三 李四" # 查出姓名含 张三、李四的
        }
    },
    "highlight":{
        "fields":{
            "name":{}
        }
    }
}

# 聚合 类似 SQL 中的 group by
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
	"aggs": {
        "all_interests":{
            "tems": {
                "field": "age" # 结果例如 : 30岁2条，20岁1条等..
            }
        }
	}
}
```

### 核心详解

#### 文档

```shell
# ES 中，Document 以 JSON 格式进行存储，可以是复杂的结构，如:
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
	"_index": "haoke",
	"_type": "user",
	"_id": "1005",
	"_version": 1,
	"_score": 1,
	"_source": {
		"id": 1005,
		"name": "孙七",
		"age": 37,
		"sex": "女",
		"card": { # 其中，card 是一个复杂对象，嵌套的 Card 对象
			"card_number": "123456789"
		}
	}
}

# 元数据<metadata>
一个文档不只有数据，还包含元数据 <关于文档的信息，三个必须的元数据节点>

_index # 文档存储所在的索引<database>
	# 事实上，数据被存储和索引在分片(shards)中，索引只是把一个或多个分片分组在一起的逻辑空间
	
_type # 文档存储所在的类<table>
	# 名字可以是大写或小写，不能包含下划线或逗号
	
_id # 文档的唯一标识
	# 与 _index 和 _type 组合时，在 ES 中即标识唯一一个 Document，可以自定义 _id，ES 生成为 32位
```

### 查询响应

#### pretty

```shell
# 在查询 url 后面添加 pretty 参数，使得返回的 json 更有可读性
```

#### 指定响应字段

```shell
# 在响应的数据中，可以指定某些需要的字段进行返回
GET http://['自己的ip 加 port']/beluga/article/1001?_source=id,name

# 如不需要返回元数据，仅仅返回原始数据
GET http://['自己的ip 加 port']/beluga/article/1001/_source

# 还可以这样
GET http://['自己的ip 加 port']/beluga/article/1001/_source?_source=id,name
```

### 判断文档是否存在

```shell
# 如果我们只需要判断文档是否存在，而不是查询文档内容
HEAD http://['自己的ip 加 port']/beluga/article/1001
```

### 批量操作

```shell
# 有些情况可以通过批量操作以减少网络请求，如: 批量查询、批量插入数据
```

#### 批量查询

```shell
POST http://['自己的ip 加 port']/beluga/article/_mget
{
    "ids":["1001","1003"]
}

# 如果某一条数据不存在，不影响整体响应，需要通过 found 的值进行判断是否查询到数据
```

#### _bulk 操作

```shell
# ES 中，支持批量的插入、修改、删除操作，都是通过 _bulk 的 api 完成的

# 请求格式如下
{ action: { metadata }}
{ request body }
{ action: { metadata }}
{ request body }
...

# 批量插入数据
{"create":{"_index":"haoke","_type":"user","_id":2001}}
{"id":2001,"name":"name1","age": 20,"sex": "男"}
{"create":{"_index":"haoke","_type":"user","_id":2002}}
{"id":2002,"name":"name2","age": 20,"sex": "男"}
{"create":{"_index":"haoke","_type":"user","_id":2003}}
{"id":2003,"name":"name3","age": 20,"sex": "男"}

# 注意最后一行回车

# 批量删除
{"delete":{"_index":"haoke","_type":"user","_id":2001}}
{"delete":{"_index":"haoke","_type":"user","_id":2002}}
{"delete":{"_index":"haoke","_type":"user","_id":2003}}
```

### 分页

```shell
# 和 SQL 使用 LIMIT 关键字返回只有一页的结果一样， ES 接受 from 和 size 参数
size # 结果数，默认 10
from # 跳过开始的结果数，默认0

GET http://['自己的ip 加 port']/beluga/article/_saerch?size=1&from=2

# 在集群系统中深度分页的问题
# 假设 5个 shard,请求结果第一页(第1个 - 第10个)，每个 shard 产生自己最顶端 10 个结果
# 并返回给请求节点，再排序这50个结果，选出顶端的10个结果。
# 假设现在请求第 1000 页，结果 10001 - 10010，每个分片都必须产生顶端的 10010 个结果
# 请求节点再排序这 50050 个结果，并丢弃掉 50040 个！
# 这也是网络搜索引擎中任何语句不能返回多于 1000 个结果的原因
```

### 映射

```shell
# 前面创建的索引以及插入数据，都是 ES 自动判断类型，有时候我们需要明确字段类型，
# 否则可能自动判断的类型与实际需求的不符。
```

#### 自动判断的规则

| JSON Type                      | Field Type |
| ------------------------------ | :--------- |
| Boolean                        | boolean    |
| Whole number                   | long       |
| Floating point                 | double     |
| String,valid date:"2017-08-15" | date       |
| String: "foo bar"              | string     |

#### Elasticsearch 支持的类型如下

| 类型           | 表示的数据类型          |
| -------------- | ----------------------- |
| String         | string,text,keyword     |
| Whole number   | byte,short,integer,long |
| Floating point | float,double            |
| Boolean        | boolean                 |
| Date           | date                    |

```shell
# string 类型在 ES 旧版本中使用较多，从 ES5.x 开始不再支持 string,由 text 和 keyword 类型替代

# text 类型
	# 当一个字段是要被全文搜索的，字段内容会被分析，在生成倒排索引以前，字符串会被分词成一个一个词项
	# text 类型的字段不用于排序，很少用于聚合
	
# keyword 类型
	# 适用于索引结构化的字段，如果字段需要进行过滤(比如查找上线的文章)、排序、聚合
	# keyword 类型的字段只能通过精确值搜索到
	
# 创建明确类型的索引
PUT http://['自己的ip 加 port']/beluga
{
    "settings": {
        "index": {
            "number_of_shards": "2",
            "number_of_replicas": "9"
        }
    },
    "mappings":{
        "user":{
            "properties":{
                "name":{
                    "type":"text"
                },
                "age":{
                    "type":"integer"
                },
                "mail":{
                    "type":"keyword"
                },
                "hobby":{
                    "type":"text"
                }
            }
        }
    }
}

# 查看映射
GET http://['自己的ip 加 port']/beluga/_mapping
```

### 结构化查询

#### term 查询

```shell
# term 主要用于精确匹配哪些值，比如数字，日期，布尔值或 not_analyzed<未经分析的文本类型> 的字符串

POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "term":{
            "id": 1001
        }
    }
}
```

#### terms 查询

```shell
# 类似 term，但允许指定多个匹配条件。如果某个字段指定了多个值，文档需要一起做匹配
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "terms":{
            "id": [1001,1002]
        }
    }
}
```

#### range 查询

```shell
# range 过滤允许我们按照指定范围查找一批数据
{
    "range":{
        "age":{
            "gte": 20,
            "lt": 30
        }
    }
}

# gt<大于> gte<大于等于> lt<小于> lte<小于等于>
```

#### exists 查询

```shell
# exists 查询可以用于查找文档中是否包含指定字段或没有某个字段，类似 SQL 的 IS_NULL
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "exists":{ # 查询出来的结果必须包含 body 这个字段
            "field":"body"
        }
    }
}
```

#### match 查询

```shell
# match 查询是一个标准查询，不管你需要全文本查询还是精确查询基本上都要用到它

# 如果你使用 match 查询一个全文本字段，它会在真正查询之前用分析器先分析 match 的字段

# 如果用 match 指定了一个确切值，在遇到数字、日期、布尔值或者 not_analyzed 的字符串时，它将为你搜索给定的值
```

#### bool 查询

```shell
# bool 查询可以用来合并多个条件查询结果的布尔逻辑，它包含以下操作符
# must 多个查询条件的完全匹配，相当于 and
# must_not 多个查询条件的相反匹配，相当于 not
# should 至少有一个查询条件匹配，相当于 or
# 这些参数可以分别继承一个查询条件或者一个查询条件的数组
{
    "bool":{
        "must": { "term": { "folder": "inbox" } },
        "must_not": { "term": {"tag": "spam" },
        "should": [
            { "term": {"starred": true} },
            { "term": {"unread": true}}
        ]
    }
}
```

### 过滤查询

```shell
# 前面讲的结构化查询，ES 也支持过滤查询，如 term、range、match 等
POST http://['自己的ip 加 port']/beluga/article/_saerch
{
    "query":{
        "bool":{
            "filter":{
                "term":{
                    "age": 20
                }
            }
        }
    }
}

# 查询和过滤的对比
	# 一条过滤语句会询问每个文档的字段值是否包含着特定值
	# 查询语句会询问每个文档的字段值与特定值的匹配程度如何
		# 一条查询语句会计算每个文档与查询语句的相关性，会给出一个相关性评分_score,并且按照相关性
		# 对匹配到的文档进行排序。
	# 一个简单的中文列表，快速匹配运算并存入内存是十分方便的，每个文档仅需要1个字节。这些缓存的过滤结果集
	# 与后续请求的结合使用是非常高效的
	# 查询语句不仅要查找相匹配的文档，还需要计算每个文档的相关性，所以一般来说，查询语句要比过滤语句更
	# 耗时，并且查询结果也不可缓存
	# 建议，做精确匹配搜索时，最好用过滤语句，因为过滤语句可以缓存数据
```

## 中文分词

```shell
# 分词是将一个文本转换成一系列单词的过程，也叫文本分析，在 ES 中称之为 Analysis
# 例如 : 我是中国人 -> 我 | 是 | 中国人
```

### 分词 API

```shell
# 指定分词器进行分词
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"standard",
    "text":"hello world"
}

# 结果中不仅可以看出分词的结果，还返回了该词在文本中的位置

# 指定索引分词
POST http://['自己的ip 加 port']/beluga/_analyze
{
    "analyzer":"standard",
    "field":"hobby",
    "text":"听音乐"
}
```

### 内置分词

#### Standard

```shell
# Standard 标准分词，按单词切分，并且会转换成小写
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"standard",
    "text": "A man becomes learned by asking questions."
}
```

#### Simple

```shell
# Simple 分词器，按照非单词切分，并且做小写处理
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"simple",
    "text":"If the document does't already exist"
}
```

#### Whitespace

```shell
# Whitespace 是按照空格切分
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"whitespace",
    "text":"If the document does't already exist"
}
```

#### Stop

```shell
# Stop 去除 Stop Word 语气助词，如 the、an 等
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"stop",
    "text":"If the document does't already exist"
}
```

#### Keyword

```shell
# keyword 分词器，意思是传入就是关键词，不做分词处理
POST http://['自己的ip 加 port']/_analyze
{
    "analyzer":"keyword",
    "text":"If the document does't already exist"
}
```

#### 中文分词

```shell
# 中文分词的难点在于，汉语中没有明显的词汇分界点

# 常用中文分词器，IK jieba THULAC 等，推荐 IK

# IK Github 站点<自定义词典扩展，禁用词典扩展等>
https://github.com/medcl/elasticsearch-analysis-ik
```

