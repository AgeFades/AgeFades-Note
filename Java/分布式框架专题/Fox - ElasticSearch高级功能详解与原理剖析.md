[TOC]

# Fox - ElasticSearch高级功能详解与原理剖析

**ES要掌握什么：**

1. **使用：搜索和聚合操作语法，理解分词，倒排索引，相关性算分（文档匹配度）**
2. **优化：数据预处理，文档建模，集群架构优化，读写性能优化**

## 1. ES数据预处理

### 1.1 Ingest Node

- ES 5.0 后，引入的一种新的节点类型。**默认配置下，每个节点都是 Ingest Node：**
  - **具有预处理数据的能力，可拦截Index或Bulk API的请求**
  - **对数据进行转换，并重新返回给Index或Bulk API**
- 无需Logstash，就可以进行数据的预处理，例如：
  - 为某个字段设置默认值
  - 重命名某个字段的字段名
  - 对字段值进行 Split 操作
  - 支持设置 Painless 脚本，对数据进行更加复杂的加工

![](https://note.youdao.com/yws/public/resource/eac9346d4648c2dce980b3c10f42a449/xmlnote/WEBRESOURCE346f78c8cf0bced05e8f59f37cda59d9/72805)

#### Ingest Node VS Logstash

|                | Logstash                                   | Ingest Node                                            |
| -------------- | ------------------------------------------ | ------------------------------------------------------ |
| 数据输入与输出 | 支持从不同的数据源读取，并写入不同的数据源 | 支持从ES REST API获取数据，并且写入Elasticsearch       |
| 数据缓冲       | 实现了简单的数据队列，支持重写             | 不支持缓冲                                             |
| 数据处理       | 支持大量的插件，也支持定制开发             | 内置的插件，可以开发Plugin进行扩展(Plugin更新需要重启) |
| 配置和使用     | 增加了一定的架构复杂度                     | 无需额外部署                                           |

### 1.2 Ingest Pipeline

- **应用场景：修复与增加写入数据**

#### 案例

- 需求：后期需要对Tags进行Aggregation统计。Tags字段中，逗号分隔的文本应该是数组，而不是一个字符串。

```bash
#Blog数据，包含3个字段，tags用逗号间隔
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}
```

#### Pipeline & Processor

- Pipeline：管道会对通过的数据（文档），按照顺序进行加工
- Processor：ES对一些加工的行为进行了抽象包装
- ES 有很多内置的 Processors，也支持通过插件的方式，实现自己的Processor
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.17/ingest-processors.html

##### 一些内置的Processors：

- Split Processor：将给定字段值分成一个数组
- Remove / Renama Processor：移除一个重命名字段
- Append：为商品增加一个新的标签
- Convert：将商品价格，从字符串转换成float类型
- Date / JSON：日期格式转换，字符串转JSON对象
- Date Index Name Processor：将通过该处理器的文档，分配到指定时间格式的索引中
- Fail Processor：一旦出现异常，该Pipeline指定的错误信息能返回给用户
- Foreach Process：数组字段，数组的每个元素都会使用到一个相同的处理器
- Grok Processor：日志的日期格式切割
- Gsub / Join / Split：字符串替换 | 数组转字符串 |字符串转数组
- Lowercase / upcase：大小写转换

![](https://note.youdao.com/yws/public/resource/eac9346d4648c2dce980b3c10f42a449/xmlnote/WEBRESOURCE09f8abc0c72966df6439c96925dd8330/72364)

```bash
# 测试split tags
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      }
    ]
  },
  "docs": [
    {
      "_index": "index",
      "_id": "1",
      "_source": {
        "title": "Introducing big data......",
        "tags": "hadoop,elasticsearch,spark",
        "content": "You konw, for big data"
      }
    },
    {
      "_index": "index",
      "_id": "2",
      "_source": {
        "title": "Introducing cloud computering",
        "tags": "openstack,k8s",
        "content": "You konw, for cloud"
      }
    }
  ]
}

#同时为文档，增加一个字段。blog查看量
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },
      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
  },

  "docs": [
    {
      "_index":"index",
      "_id":"1",
      "_source":{
        "title":"Introducing big data......",
        "tags":"hadoop,elasticsearch,spark",
        "content":"You konw, for big data"
      }
    },
    {
      "_index":"index",
      "_id":"2",
      "_source":{
        "title":"Introducing cloud computering",
        "tags":"openstack,k8s",
        "content":"You konw, for cloud"
      }
    }

    ]
}
```

##### 创建Pipeline

```bash
# 为ES添加一个 Pipeline
PUT _ingest/pipeline/blog_pipeline
{
  "description": "a blog pipeline",
  "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },

      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
}

#查看Pipleline
GET _ingest/pipeline/blog_pipeline
```

#####  使用Pipeline更新数据

```bash

#不使用pipeline更新数据
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
}

#使用pipeline更新数据
PUT tech_blogs/_doc/2?pipeline=blog_pipeline
{
  "title": "Introducing cloud computering",
  "tags": "openstack,k8s",
  "content": "You konw, for cloud"
}
```

##### 借助update_by_query更新已存在的文档

```bash
#update_by_query 会导致错误
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{
}

#增加update_by_query的条件
POST tech_blogs/_update_by_query?pipeline=blog_pipeline
{
    "query": {
        "bool": {
            "must_not": {
                "exists": {
                    "field": "views"
                }
            }
        }
    }
}

GET tech_blogs/_search
```

### 1.3 Painless Script

- 自ES 5.x后引入，专门为ES设计，扩展了Java的语法。
- Painless支持所有Java的数据类型及Java API子集
- Painless Script具备如下特性：
  - 高性能/安全
  - 支持显示类型或者动态语义类型

#### Painless的用途

- **可以对文档字段进行加工处理**
  - 更新或删除字段，处理数据聚合操作
  - Script Field：对返回的字段提前进行计算
  - Function Score：对文档的算分进行处理
- **在Ingest Pipeline中执行脚本**
- **在Reindex API，Update By Query时，对数据进行处理**

#### 通过Painless脚本访问字段

| 上下文               | 语法                   |
| -------------------- | ---------------------- |
| Ingestion            | ctx.field_name         |
| Update               | ctx._source.field_name |
| Search & Aggregation | doc["field_name"]      |

- 测试

```bash
# 增加一个 Script Prcessor
POST _ingest/pipeline/_simulate
{
  "pipeline": {
    "description": "to split blog tags",
    "processors": [
      {
        "split": {
          "field": "tags",
          "separator": ","
        }
      },
      {
        "script": {
          "source": """
          if(ctx.containsKey("content")){
            ctx.content_length = ctx.content.length();
          }else{
            ctx.content_length=0;
          }

          """
        }
      },

      {
        "set":{
          "field": "views",
          "value": 0
        }
      }
    ]
  },

  "docs": [
    {
      "_index":"index",
      "_id":"1",
      "_source":{
        "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data"
      }
    },


    {
      "_index":"index",
      "_id":"2",
      "_source":{
        "title":"Introducing cloud computering",
  "tags":"openstack,k8s",
  "content":"You konw, for cloud"
      }
    }

    ]
}

DELETE tech_blogs
PUT tech_blogs/_doc/1
{
  "title":"Introducing big data......",
  "tags":"hadoop,elasticsearch,spark",
  "content":"You konw, for big data",
  "views":0
}

POST tech_blogs/_update/1
{
  "script": {
    "source": "ctx._source.views += params.new_views",
    "params": {
      "new_views":100
    }
  }
}

# 查看views计数
POST tech_blogs/_search



#保存脚本在 Cluster State
POST _scripts/update_views
{
  "script":{
    "lang": "painless",
    "source": "ctx._source.views += params.new_views"
  }
}

POST tech_blogs/_update/1
{
  "script": {
    "id": "update_views",
    "params": {
      "new_views":1000
    }
  }
}


GET tech_blogs/_search
{
  "script_fields": {
    "rnd_views": {
      "script": {
        "lang": "painless",
        "source": """
          java.util.Random rnd = new Random();
          doc['views'].value+rnd.nextInt(1000);
        """
      }
    }
  },
  "query": {
    "match_all": {}
  }
}
```

## 2. ES文档建模

### 2.1 ES中如何处理关联关系

- 关系型数据库范式化（Normalize）设计的主要目标是减少不必要的更新，往往会带来一些副作用
  - 一个完全范式化设计的数据库会经常面临 “查询缓慢” 的问题。数据库约范式化，就需要Join越多的表
  - 范式化节省了存储空间，但是存储空间已经变得越来越便宜
  - 范式化简化了更新，但是数据读取操作可能更多
- **反范式化（Denormalize）的设计不使用关联关系，而是在文档中保存冗余的数据拷贝。**
  - **优点：无需处理Join操作，数据读取性能好。**ES可以通过压缩_source字段，减少磁盘空间的开销
  - **缺点：不适合在数据频繁修改的场景。**一条数据的改动，可能会引起很多数据的更新



- ES并不擅长处理关联关系，一般会采用以下四种方法处理关联：
  - **对象类型**
  - **嵌套对象（Nested Object）**
  - **父子关联关系（Parent / Child）**
  - **应用端关联**

### 2.2 对象类型

