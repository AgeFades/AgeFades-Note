[TOC]

# Fox - MongoDB聚合操作及索引使用详解

## 一、聚合操作

- **聚合操作处理数据记录并返回计算结果**
  - 聚合操作组值来自多个文档，可以对分组数据执行各种操作以返回单个结果。
- **聚合操作包含三类**：
  - 单一作用聚合
    - 提供了对常见聚合过程的简单访问，操作都从单个集合聚合文档
  - 聚合管道
    - **是一个数据聚合的框架，模型基于数据处理流水线的概念。**
    - 文档进入多级管道，将文档转换为聚合结果
  - MapReduce
    - MapReduce操作具有两个阶段：
      - 处理每个文档并向每个输入文档发射一个或多个对象的map阶段
      - 以及reduce组合map操作的输出阶段

### 1.1 单一作用聚合

- MongoDB提供一类单一作用的聚合函数，类似于：

| db.collection.estimatedDocumentCount() | 返回集合或视图中所有文档的计数                               |
| -------------------------------------- | ------------------------------------------------------------ |
| db.collection.count()                  | 返回与find()集合或视图的查询匹配的文档计数 。等同于 db.collection.find(query).count()构造 |
| db.collection.distinct()               | 在单个集合或视图中查找指定字段的不同值，并在数组中返回结果。 |

- 所有这些操作都聚合来自单个集合的文档。
- 虽然这些操作提供了对公共聚合过程的简单访问，但它们缺乏聚合管道和MapReduce的灵活性和功能

![](https://note.youdao.com/yws/public/resource/dad1b8090503bafd9cc48019bda7570f/xmlnote/8FE5C77DF25949E9859FB70A27338EC0/39657)

- 注意：在分片集群下，如果存在孤立文档或正在进行块迁移，
  - 则 db.collection.count() 没有查询谓词可能导致计数不正确。
  - 要避免这些情况，需在分片集群上使用 db.collection.aggregate() 方法
- 使用示例如下：

```javascript
// 检索 books 集合中所有文档的计数
db.books.estimatedDocumentCount()
// 计算与查询匹配的所有文档数
db.books.count({favCount:{$gt:50}})
// 返回不同type的数组
db.books.distinct("type")
// 返回收藏数大于90的文档不同type的数组
db.books.distinct("type", {favCount: {$gt:90}})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652327171124.png)

### 1.2 聚合管道

- **MongoDB聚合框架（Aggregation Framework）是一个计算框架**，它可以：
  - **作用在一个或几个集合上**
  - **对集合中的数据进行的一系列运算**
  - **将这些数据转化为期望的形式**
- 从效果而言，聚合框架相当于SQL查询中的 group by、left outer join、as 等

#### 管道（Pipeline）和阶段（Stage）

- **整个聚合运算过程成为管道（Pipeline），它是由多个阶段（Stage）组成的**，每个管道：
  - 接受一系列文档（原始数据）
  - 每个阶段对这些文档进行一系列运算
  - 结果文档输出给下一个阶段

![](https://note.youdao.com/yws/public/resource/dad1b8090503bafd9cc48019bda7570f/xmlnote/731EA7E78D384C938C9221C99BDB5FB4/39636)

##### 聚合管道操作语法

```javascript
pipeline = [$stage1, $stage2, ...$stageN];
db.collecion.aggregate(pipeline, {options})
```

- pipelines 一组数据聚合阶段，除 $out、$Merge 和 $geonear 阶段之外，每个阶段都可以在管道中出现多次
- options 可选，是聚合操作的其他参数，包括：
  - 查询计划
  - 是否使用临时文件
  - 游标
  - 最大操作时间
  - 读写策略
  - 强制索引
  - ...

![](https://note.youdao.com/yws/public/resource/dad1b8090503bafd9cc48019bda7570f/xmlnote/F64F9F488D494D7BAA2983FEB6A68C37/39613)

##### 常用的管道聚合阶段

- 聚合管道包含非常丰富的聚合阶段，下面是最常用的聚合阶段

| 阶段           | 描述     | SQL等价运算符   |
| -------------- | -------- | --------------- |
| $match         | 筛选条件 | WHERE           |
| $project       | 投影     | AS              |
| $lookup        | 左外连接 | LEFT OUTER JOIN |
| $sort          | 排序     | ORDER BY        |
| $group         | 分组     | GROUP BY        |
| $skip/$limit   | 分页     |                 |
| $unwind        | 展开数组 |                 |
| $graphLookup   | 图搜索   |                 |
| $facet/$bucket | 分面搜索 |                 |

[官方管道聚合文档](https://docs.mongodb.com/manual/reference/operator/aggregation-pipeline/)

##### 聚合表达式

###### 获取字段信息

```shell
$<field>  ： 用 $ 指示字段路径
$<field>.<sub field>  ： 使用 $  和 .  来指示内嵌文档的路径
```

###### 常量表达式

```shell
$literal :<value> ： 指示常量 <value>
```

###### 系统变量表达式

```shell
$$<variable>  使用 $$ 指示系统变量
$$CURRENT  指示管道中当前操作的文档
```

#### 操作示例

##### 数据准备

- 宿主机 vim book.js

```javascript
var tags = ["nosql","mongodb","document","developer","popular"];
var types = ["technology","sociality","travel","novel","literature"];
var books=[];
for(var i=0;i<50;i++){
    var typeIdx = Math.floor(Math.random()*types.length);
    var tagIdx = Math.floor(Math.random()*tags.length);
    var tagIdx2 = Math.floor(Math.random()*tags.length);
    var favCount = Math.floor(Math.random()*100);
    var username = "xx00"+Math.floor(Math.random()*10);
    var age = 20 + Math.floor(Math.random()*15);
    var book = {
        title: "book-"+i,
        type: types[typeIdx],
        tag: [tags[tagIdx],tags[tagIdx2]],
        favCount: favCount,
        author: {name:username,age:age}
    };
    books.push(book)
}
db.books.insertMany(books);
```

- mongo shell 中执行

```javascript
load("book.js")
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334258952.png)

##### $project

- **投影操作**，将原始字段投影成指定名称，如将集合中的title投影成name

```javascript
db.books.aggregate([{$project:{name:"$title"}}])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334347241.png)

- $project 可以灵活控制输出文档的格式，也可以剔除不需要的字段

```javascript
db.books.aggregate([{$project:{name:"$title",_id:0,type:1,author:1}}])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334423453.png)

- 从嵌套文档中排除字段

```javascript
db.books.aggregate([
    {$project:{name:"$title",_id:0,type:1,"author.name":1}}
])
或者
db.books.aggregate([
    {$project:{name:"$title",_id:0,type:1,author:{name:1}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334492605.png)

##### $match

- **$match用于对文档进行筛选**，之后可以在得到的文档子集上做聚合
  - $match可以使用除了地理空间之外的所有常规查询操作符，
  - **在实际应用中尽可能将$match放在管道的前面位置**
    - 一是可以快速将不需要的文档过滤掉，以减少管道的工作量
    - 二是如果在投射和分组之前执行$match，查询可以使用索引

```javascript
db.books.aggregate([{$match:{type:"technology"}}])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334670875.png)

```javascript
db.books.aggregate([
    {$match:{type:"technology"}},
    {$project:{name:"$title",_id:0,type:1,author:{name:1}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334736823.png)

##### $count

- 计数并返回与查询匹配的结果数
  - $match筛选出 type=technology 的文档，并传到下一阶段
  - $count返回聚合管道中剩余文档的计数，并将该值分配给 type_count

```javascript
db.books.aggregate([
    {$match:{type:"technology"}},
    {$count: "type_count"}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652334827187.png)

##### $group

- **按指定的表达式对文档进行分组，并将每个不同分组的文档输出到下一个阶段**
  - 输出文档包含一个 _id 字段，该字段按键包含不同的组
- 输出文档还可以包含计算字段，该字段保存由$group中的_id字段分组的一些accumulator表达式的值
  - **$group不会输出具体的文档而只是统计信息**

```javascript
{ $group: { _id: <expression>, <field1>: { <accumulator1> : <expression1> }, ... } }
```

- _id字段是必填的，但是，可以指定 _id 值为null来为整个输入文档计算累计值
- 剩余的计算字段是可选的，并使用 <accumulator> 运算符来进行计算
- _id和<accumulator>表达式可以接受任何有效的 [表达式](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/#aggregation-expressions)

###### accumulator操作符

| 名称        | 描述                                                         | 类比sql   |
| ----------- | ------------------------------------------------------------ | --------- |
| $avg        | 计算均值                                                     | avg       |
| $first      | 返回每组第一个文档，如果有排序，按照排序，如果没有按照默认的存储的顺序的第一个文档。 | limit 0,1 |
| $last       | 返回每组最后一个文档，如果有排序，按照排序，如果没有按照默认的存储的顺序的最后个文档。 | -         |
| $max        | 根据分组，获取集合中所有文档对应值得最大值。                 | max       |
| $min        | 根据分组，获取集合中所有文档对应值得最小值。                 | min       |
| $push       | 将指定的表达式的值添加到一个数组中。                         | -         |
| $addToSet   | 将表达式的值添加到一个集合中（无重复值，无序）。             | -         |
| $sum        | 计算总和                                                     | sum       |
| $stdDevPop  | 返回输入值的总体标准偏差（population standard deviation）    | -         |
| $stdDevSamp | 返回输入值的样本标准偏差（the sample standard deviation）    | -         |

- **$group阶段的内存限制为100M。默认情况下，如果stage超过此限制，$group将产生错误**
  - 要允许处理大型数据集时，需将 **allowDiskUse** 选项设置为true 以启用$group操作以写入临时文件

###### 操作示例

```javascript
// 查book的数量，收藏总数和平均值
db.books.aggregate([
    {$group:{_id:null,count:{$sum:1},pop:{$sum:"$favCount"},avg:{$avg:"$favCount"}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652335418449.png)

```javascript
// 统计每个作者的book收藏总数
db.books.aggregate([
    {$group:{_id:"$author.name",pop:{$sum:"$favCount"}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652335491729.png)

```javascript
// 统计每个作者的每本book的收藏数
db.books.aggregate([
    {$group:{_id:{name:"$author.name",title:"$title"},pop:{$sum:"$favCount"}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652335559201.png)

```javascript
// 每个作者的book的type合集
db.books.aggregate([
    {$group:{_id:"$author.name",types:{$addToSet:"$type"}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652335653059.png)

##### $unwind

- **可以将数组拆分为单独的文档**
- v3.2+支持如下语法：

```shell
{
  $unwind:
    {
      # 要指定字段路径，在字段名称前加上$符并用引号括起来。
      path: <field path>,
      # 可选,一个新字段的名称用于存放元素的数组索引。该名称不能以$开头。
      includeArrayIndex: <string>,  
      # 可选，default :false，若为true,如果路径为空，缺少或为空数组，则$unwind输出文档
      preserveNullAndEmptyArrays: <boolean> 
 		} 
 }
```

- 姓名为xx006的作者的book的tag数组拆分为多个文档

```javascript
db.books.aggregate([
    {$match:{"author.name":"xx006"}},
    {$unwind:"$tag"}
])

db.books.aggregate([
    {$match:{"author.name":"xx006"}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652336004136.png)

- 每个作者的book的tag合集

```javascript
db.books.aggregate([
    {$unwind:"$tag"},
    {$group:{_id:"$author.name",types:{$addToSet:"$tag"}}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652336142011.png)

###### 案例

- 示例数据

```javascript
db.books.insert([
{
	"title" : "book-51",
	"type" : "technology",
	"favCount" : 11,
     "tag":[],
	"author" : {
		"name" : "fox",
		"age" : 28
	}
},{
	"title" : "book-52",
	"type" : "technology",
	"favCount" : 15,
	"author" : {
		"name" : "fox",
		"age" : 28
	}
},{
	"title" : "book-53",
	"type" : "technology",
	"tag" : [
		"nosql",
		"document"
	],
	"favCount" : 20,
	"author" : {
		"name" : "fox",
		"age" : 28
	}
}])
```

- 测试

```javascript
// 使用 includeArrayIndex 选项来输出数组元素的数组索引
db.books.aggregate(
	[
    {$match: {"author.name":"fox"}},
    {$unwind: {path:"$tag", includeArrayIndex: "arrayIndex"}}
  ]
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652336308492.png)

```javascript
// 使用 preserveNullAndEmptyArrays 选项在输出中包含缺少 size 字段，null或空数组的文档
db.books.aggregate([
    {$match:{"author.name":"fox"}},
    {$unwind:{path:"$tag", preserveNullAndEmptyArrays: true}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652336421807.png)

##### $limit

- 限制传递到管道中下一个阶段的文档数

```javascript
db.books.aggregate([
    {$limit : 5 }
])
```

- 此操作仅返回管道传递给它的前5个文档，对文档内容本身没有影响
- **注意：当 $sort 在管道中的 $limit 之前立即出现时，$sort 操作只会在过程中维持前n个结果**
  - 其中 n 是指定的限制，而MongoDB只需要将n个选项存储在内存中

##### $skip

- 跳过进入stage的指定数量的文档，并将其余文档传递到管道中的下一个阶段

```javascript
db.books.aggregate([
    {$skip : 5 }
])
```

- 此操作将跳过管道传递给它的前5个文档。$skip对沿着管道传递的文档的内容没有影响。

##### $sort

- 对所有输入文档进行排序，并按排序顺序将它们返回到管道，语法如下：

```javascript
{ $sort: { <field1>: <sort order>, <field2>: <sort order> ... } }
```

- 要对字段进行排序，需将排序顺序设置为1（升序）或-1（降序），如下例所示：

```javascript
// 先按 favCount 降序，favCount相同时，按 title 升序
db.books.aggregate([
    {$sort : {favCount:-1,title:1}}
])
```

##### $lookup

- MongoDB 3.2版本新增，主要用来实现多表关联查询
- **每个输入待处理的文档，经过$lookup阶段的处理，输出的新文档中会包含一个新生成的数组（可根据需要命名新key）**
- 数组列存放的数据是来自被 join 集合的适配文档，如果没有，集合为空（即为 [ ]）
- 语法如下：

```javascript
db.collection.aggregate([{
      $lookup: {
             from: "<collection to join>",
             localField: "<field from the input documents>",
             foreignField: "<field from the documents of the from collection>",
             as: "<output array field>"
           }
  })
```

| from         | 同一个数据库下等待被Join的集合。                             |
| ------------ | ------------------------------------------------------------ |
| localField   | 源集合中的match值，如果输入的集合中，某文档没有 localField这个Key（Field），在处理的过程中，会默认为此文档含有 localField：null的键值对。 |
| foreignField | 待Join的集合的match值，如果待Join的集合中，文档没有foreignField值，在处理的过程中，会默认为此文档含有 foreignField：null的键值对。 |
| as           | 为输出文档的新增值命名。如果输入的集合中已存在该值，则会覆盖掉 |

- **注意：null = null 此为真**
- 其语法功能类似于下面的伪SQL语句

```sql
SELECT *, <output array field>
FROM collection
WHERE <output array field> IN (SELECT *
                               FROM <collection to join>
                               WHERE <foreignField>= <collection.localField>);
```

###### 案例

- 数据准备

```javascript
db.customer.insert({customerCode:1,name:"customer1",phone:"13112345678",address:"test1"})
db.customer.insert({customerCode:2,name:"customer2",phone:"13112345679",address:"test2"})

db.order.insert({orderId:1,orderCode:"order001",customerCode:1,price:200})
db.order.insert({orderId:2,orderCode:"order002",customerCode:2,price:400})

db.orderItem.insert({itemId:1,productName:"apples",qutity:2,orderId:1})
db.orderItem.insert({itemId:2,productName:"oranges",qutity:2,orderId:1})
db.orderItem.insert({itemId:3,productName:"mangoes",qutity:2,orderId:1})
db.orderItem.insert({itemId:4,productName:"apples",qutity:2,orderId:2})
db.orderItem.insert({itemId:5,productName:"oranges",qutity:2,orderId:2})
db.orderItem.insert({itemId:6,productName:"mangoes",qutity:2,orderId:2})
```

- 关联查询

```javascript
db.customer.aggregate([        
    {$lookup: {
       from: "order",
       localField: "customerCode",
       foreignField: "customerCode",
       as: "customerOrder"
     }
    } 
])

db.order.aggregate([
    {$lookup: {
               from: "customer",
               localField: "customerCode",
               foreignField: "customerCode",
               as: "curstomer"
             }
        
    },
    {$lookup: {
               from: "orderItem",
               localField: "orderId",
               foreignField: "orderId",
               as: "orderItem"
             }
    }
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652338083874.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652338247713.png)

#### 聚合操作案例1

- 统计每个分类的book文档数量

```javascript
db.books.aggregate([
    {$group:{_id:"$type",total:{$sum:1}}},
    {$sort:{total:-1}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652338481593.png)

- 标签的热度排行，标签的热度则按其关联book文档的收藏数（favCount）来计算
  - $match阶段：用于过滤favCount=0的文档
  - $unwind阶段：用于将标签数组进行展开，这样一个包含3个标签的文档会被拆解为3个条目
  - $group阶段：对拆解后的文档进行分组计算，$sum：$favCount 表示对 favCount 值求和
  - $sort阶段：接收分组计算的输出，按 total 得分进行降序

```javascript
db.books.aggregate([
    {$match:{favCount:{$gt:0}}},
    {$unwind:"$tag"},
    {$group:{_id:"$tag",total:{$sum:"$favCount"}}},
    {$sort:{total:-1}}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652338980799.png)

- 统计book文档收藏数 **[0,10),[10,60),[60,80),[80,100),[100,+∞）**

```javascript
db.books.aggregate([{
    $bucket:{
        groupBy:"$favCount",
        boundaries:[0,10,60,80,100],
        default:"other",
        output:{"count":{$sum:1}}
    }
}])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652339317957.png)

#### 聚合操作案例2

- 导入 [邮政编码数据集](https://media.mongodb.org/zips.json)

##### 使用mongoimport工具导入数据

- mongo bin 目录本身没有 mongoimport 命令
- 需在 [官方Mongo工具集下载](https://www.mongodb.com/try/download/database-tools)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652340360292.png)

- 下载后上传到mongo所在宿主机、解压、复制bin下可执行命令到 mongo的bin目录下

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652340464630.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652340566213.png)

- 使用命令导入数据
  - --host：代表远程连接的数据库地址，默认链接本地
  - --port：代表远程连接的数据库端口，默认是27017
  - -u、--username：代表连接数据库的账号，如果设置数据库的认证，需要指定用户账号
  - -p、--password：代表连接数据库账号对应的密码
  - -d、--db：代表连接的数据库
  - -c、--collection：代表连接数据库中的集合
  - -f、--fields：代表导入集合中的字段
  - --type：代表导入的文件类型，包括csv和json,tsv文件，默认json格式
  - --file：导入的文件名称
  - --headerline：导入csv文件时，指明第一行是列名，不需要导入

```shell
mongoimport -d test -u agefades -p agefades --authenticationDatabase=admin -c zips --file zips.json
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652340623465.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652341141737.png)

##### 测试案例

- 返回人口超过1000万的州

```javascript
db.zips.aggregate(
	[
    {
      $group: {_id: "$state", totalPop: {$sum: "$pop"}}
    },
    {
      $match: {totalPop: {$gte: 10*1000*1000}}
    }
  ]
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652341276727.png)

- 这个聚合操作的等价SQL是：

```sql
SELECT state, SUM(pop) AS totalPop
FROM zips
GROUP BY state
HAVING totalPop >= (10*1000*1000)
```

- 返回各州平均城市人口

```javascript
db.zips.aggregate( [
   { $group: { _id: { state: "$state", city: "$city" }, cityPop: { $sum: "$pop" } } },
   { $group: { _id: "$_id.state", avgCityPop: { $avg: "$cityPop" } } }
] )

db.zips.aggregate( [
   { $group: { _id: { state: "$state", city: "$city" }, cityPop: { $sum: "$pop" } } } 
] )
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652343035307.png)

- 按州返回最大和最小的城市

```javascript
db.zips.aggregate( [
   { $group:
      {
        _id: { state: "$state", city: "$city" },
        pop: { $sum: "$pop" }
      }
   },
   { $sort: { pop: 1 } },
   { $group:
      {
        _id : "$_id.state",
        biggestCity:  { $last: "$_id.city" },
        biggestPop:   { $last: "$pop" },
        smallestCity: { $first: "$_id.city" },
        smallestPop:  { $first: "$pop" }
      }
   },
  { $project:
    { _id: 0,
      state: "$_id",
      biggestCity:  { name: "$biggestCity",  pop: "$biggestPop" },
      smallestCity: { name: "$smallestCity", pop: "$smallestPop" }
    }
  }
] )
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652343237804.png)

### 1.3 MapReduce

- MapReduce操作将大量的数据处理工作，划分成多个线程并行处理，然后将结果合并在一起
  - MongoDB提供的 MapReduce 非常灵活，对于大规模数据分析也相当实用
- MapReduce具有两个阶段
  1. **将具有相同Key的文档数据整合在一起的map阶段**
  2. **组合map操作的结果进行统计输出的reduce阶段**
- MapReduce的基本语法：
  - map：将数据拆分成键值对，交给reduce函数
  - reduce：根据键将值做统计运算
  - out：可选，将结果汇入指定表
  - query：可选，筛选数据的条件，符合条件的数据送入map
  - sort：排序完后，送入map
  - limit：限制送入map的文档数
  - finalize：可选，修改 reduce 的结果后进行输出
  - scope：可选，指定map、reduce、finalize的全局变量
  - jsMode：可选，默认false。在MapReduce过程中，是否将数据转换成 BSON 格式
  - verbose：可选，是否在结果中显示时间，默认false
  - bypassDocumentValidation：可选，是否略过数据校验

```javascript
db.collection.mapReduce(
   function() {emit(key,value);},  //map 函数
   function(key,values) {return reduceFunction},   //reduce 函数
   {
      out: <collection>,
      query: <document>,
      sort: <document>,
      limit: <number>,
     finalize: <function>, 
     scope: <document>,
     jsMode: <boolean>,
     verbose: <boolean>,
     bypassDocumentValidation: <boolean>
   }
)
```

![](https://note.youdao.com/yws/public/resource/dad1b8090503bafd9cc48019bda7570f/xmlnote/E57D4B5E5A6E4CE1AD3079772A7688A6/39652)

#### 测试案例

- 统计 type为travel 的不同作者的book文档收藏数

```javascript
db.books.mapReduce(
	function() {
    emit(this.author.name, this.favCount)
  },
  function(key, values) {
    return Array.sum(values)
  },
  {
    query: {type: "travel"},
    out: "books_favCount"
  }
)

db.books_favCount.find()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652344502577.png)

#### 版本问题

- **从MongoDB5.0开始，MapReduce操作已被弃用**
  - 聚合管道比 MapReduce 提供更好的性能和可用性
  - MapReduce操作可以使用聚合管道操作符重写，例如$group、$merge等

## 二、MongoDB索引

- **索引是一种用来快速查询数据的数据结构**
- B+Tree就是一种常用的数据库索引数据结构，**MongoDB采用B+Tree做索引**
- 索引创建在 collections 上。

### 2.1 MongoDB索引数据结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652346843555.png)

### 2.2 WiredTiger数据文件在磁盘的存储结构

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/998EDAD804F04A488345D84C6CF68957/42537)

- B+Tree中的 left page 包含一个 页头(page header)、块头(block header) 和真正的数据(key/value)
  - 页头：定义了页的类型、页中实际载荷数据的大小、页中记录条数等信息
  - 块头：定义了此页的checksum、块在磁盘上的寻址位置等信息
- WiredTiger 有一个块设备管理的模块，用来为 page 分配 block。
  - 如果要定位某一行数据（key/value）的位置，可以先通过block的位置找到此page（相对于文件起始位置的偏移量）
  - 再通过page找到行数据的相对位置，最后可以得到行数据相对于文件起始位置的偏移量offsets

### 2.3 索引的分类

- 按照索引包含的字段数量，可以分为：
  - 单建索引
  - 组合索引（复合索引）
- 按照索引字段的类型，可以分为：
  - 主键索引
  - 非主键索引
- 按照索引节点与物理记录的对应方式来分，可以分为：
  - 聚簇索引（索引节点上直接包含了数据记录）
  - 非聚簇索引（索引节点上仅仅包含了一个指向数据记录的指针）
- 按照索引的特性不同，可以分为：
  - 唯一索引
  - 稀疏索引
  - 文本索引
  - 地理空间索引
  - ...



- 与大多数DB一样，MongoDB支持各种丰富的索引类型
  - 由于采用了灵活可变的文档类型，也支持对嵌套字段、数组进行索引
  - 在一些特殊应用场景，MongoDB还支持地理空间索引、文本检索索引、TTL索引等不同的特性

### 2.3 索引设计原则

1. 每个查询原则上都需要创建对应索引
2. 单个索引设计应考虑满足尽量多的查询
3. 索引字段选择及顺序需要考虑查询覆盖率及选择性
4. 对于更新极其频繁的字段，创建索引需慎重
5. 对于数组索引需要慎重考虑未来元素个数
6. 对于超长字符串类型字段，慎用索引
7. 并发更新较高的单个集合上不宜创建过多索引

### 2.4 索引操作

#### 创建索引

- 语法如下：

```apl
db.collection.createIndex(keys, options)
```

- Key值为需要创建的索引字段，1按升序创建索引，-1按降序创建索引
- 可选参数列表如下：

| **Parameter**      | **Type**      | **Description**                                              |
| ------------------ | ------------- | ------------------------------------------------------------ |
| 2                  | Boolean       | 建索引过程会阻塞其它数据库操作，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为false。 |
| unique             | Boolean       | 建立的索引是否唯一。指定为true创建唯一索引。默认值为false.   |
| name               | string        | 索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称。 |
| dropDups           | Boolean       | 3.0+版本已废弃。在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 false. |
| sparse             | Boolean       | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档。默认值为 false. |
| expireAfterSeconds | integer       | 指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。 |
| v                  | index version | 索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。 |
| weights            | document      | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。 |
| default_language   | string        | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语 |
| language_override  | string        | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language. |

- 注意：3.0.0 版本前创建索引方法为 db.collection.ensureIndex()

##### 示例

```shell
# 创建索引后台执行
db.values.createIndex({open: 1, close: 1}, {background: true})
# 创建唯一索引
db.values.createIndex({title:1},{unique:true})
```

#### 查看索引

##### 示例

```shell
# 查看索引信息
db.books.getIndexes()

# 查看索引键
db.books.getIndexKeys()

# 查看索引占用空间
	# is_detail：可选参数
		# 传0或false以外的任意数据，都会显示该集合中每个索引的大小及总大小
		# 传0或false(默认)，只显示该集合中的所有索引的总大小
db.collection.totalIndexSize([is_detail])
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/8549B26B6C774792A5C506909F70C5F4/39499)

#### 删除索引

##### 示例

```shell
# 删除集合指定索引
db.col.dropIndex("索引名称")

# 删除集合所有索引，不能删除主键索引
db.col.dropIndexes()
```

### 2.5 索引类型

#### 单键索引（Single Field Indexs）

- 在某个特定字段上建立索引
- MongoDB在 _id 上建立了唯一的单键索引，所以经常使用 _id 来进行查询
- 在索引字段上进行精确匹配、排序、范围查找，都可以使用该索引

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/851B5AB741A94684872F0CBED22FF808/39747)

##### 案例

```apl
db.books.createIndex({title:1})
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/ABAD8D510ECA433A8C2FB63803F1417A/39472)

- 对内嵌文档字段创建索引

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/895CE60E419E4062BBFA2F7F46E7CC21/39469)

#### 复合索引（Compound Index）

- 由多个字段组合而成的索引。性质与单键索引类似
  - 不同的是，**复合索引中字段的顺序、字段的升降序对查询性能有直接的影响**
  - 因此，在设计复合索引时，需要考虑不同的查询场景

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/5CB6D5DD151840AD8FDC8C6E75E13DEF/39749)

##### 案例

```apl
db.books.createIndex({type:1,favCount:1})
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/301728F6622C4E0A81BE8E6A6041CEB4/39474)

#### 多键索引（Multikey Index）

- **在数组的属性上建立索引**
  - 针对这个数组的任意值的查询都会定位到这个文档
  - 即多个索引入口或者键值引用同一个文档
- 多键索引很容易与复合索引产生混淆
  - **复合索引是多个字段的组合，而多键索引仅仅是在一个字段上出现了多键**
  - 实际上，多键索引也可以出现在复合索引里
  - **注意，MongoDB并不支持一个复合索引中同时出现多个数组字段**

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/C2A1D87134F54F7B96FB17140F6C404A/39752)

##### 案例

- 嵌入文档的索引数组

```apl
db.inventory.insertMany([
{
  _id: 1,
  item: "abc",
  stock: [
    { size: "S", color: "red", quantity: 25 },
    { size: "S", color: "blue", quantity: 10 },
    { size: "M", color: "blue", quantity: 50 }
  ]
},
{
  _id: 2,
  item: "def",
  stock: [
    { size: "S", color: "blue", quantity: 20 },
    { size: "M", color: "blue", quantity: 5 },
    { size: "M", color: "black", quantity: 10 },
    { size: "L", color: "red", quantity: 2 }
  ]
},
{
  _id: 3,
  item: "ijk",
  stock: [
    { size: "M", color: "blue", quantity: 15 },
    { size: "L", color: "blue", quantity: 100 },
    { size: "L", color: "red", quantity: 25 }
  ]
}
])
```

- 在包含嵌套的数组字段上创建多键索引

```apl
db.inventory.createIndex( {"stock.size": 1, "stock.quantity": 1} )

db.inventory.find( {"stock.size": "S", "stock.quantity": {$gt: 20}} )
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/5FE966F9E55C4E64BADFE84A516731E7/39542)

#### 地理空间索引（Geospatial Index）

- 移动互联网时代，基于地理位置的检索（LBS）功能几乎是所有应用系统的标配。
- MongoDB为地理空间检索提供了非常方便的功能。
- **地理空间索引（2dsphere index）就是专门用于实现位置检索的一种特殊索引**

##### 案例

- MongoDB实现 **查询附近商家**
- 假设商家数据模型如下：

```apl
db.restaurant.insert({
    restaurantId: 0,
    restaurantName:"兰州牛肉面",
    location : {
        type: "Point",
        coordinates: [ -73.97, 40.77 ]
    }
})
```

- 创建一个2dsphere索引

```apl
db.restaurant.createIndex({location : "2dsphere"})
```

- 查询附近 10km 商家信息

```apl
db.restaurant.find( { 
    location:{ 
        $near :{
            $geometry :{ 
                type : "Point" ,
                coordinates : [  -73.88, 40.78 ] 
            } ,
            $maxDistance:10000 
        } 
    } 
} )
```

- $near 查询操作符，用于实现附近商家的检索，返回数据结果会按距离排序
- $geometry 操作符，用于指定一个 GeoJSON  格式的地理空间对象
  - type=Point 表示地理坐标点
  - coordinates 是用户当前所在的经纬度位置
  - $maxDistance 限定了最大距离，单位是米

#### 全文索引（Text Indexes）

- MongoDB支持全文检索功能，可通过建立文本索引来实现简易的分词检索
  - 文本索引存在诸多限制，官方并未提供中文分词的功能

##### 案例

- 创建索引

```apl
db.reviews.createIndex( { comments: "text" } )
```

- **$text 操作符可以在有 text index 的集合上执行文本检索**
  - $text将会使用空格和标点符号作为分隔符对检索字符串进行分词，
  - 并对检索字符串中所有的分词结果进行一个逻辑上的 OR 操作

- 数据准备如下：

```apl
db.stores.insert(
   [
     { _id: 1, name: "Java Hut", description: "Coffee and cakes" },
     { _id: 2, name: "Burger Buns", description: "Gourmet hamburgers" },
     { _id: 3, name: "Coffee Shop", description: "Just coffee" },
     { _id: 4, name: "Clothes Clothes Clothes", description: "Discount clothing" },
     { _id: 5, name: "Java Shopping", description: "Indonesian goods" }
   ]
)
```

- 创建name和description的全文索引

```apl
db.stores.createIndex({name: "text", description: "text"})
```

- 通过$text操作符查数据

```apl
db.stores.find({$text: {$search: "java coffee shop"}})
```

#### Hash索引（Hashed Indexes）

- 不同于传统的 B-Tree 索引，Hash索引使用 Hash函数 来创建索引
- **在索引字段上进行精确匹配，但不支持范围查询，不支持多键hash**
- Hash索引上的入口是均匀分布的，在分片集合中非常有用

```apl
db.users.createIndex({username : 'hashed'})
```

#### 通配符索引（Wildcard Indexes）

- MongoDB的文档模式是动态变化的，而通配符索引可以建立在一些不可预知的字段上，以实现查询的加速
  - **MongoDB4.2 引入了通配符索引以支持对未知或任意字段的查询**

##### 案例

- 准备商品数据，不同商品属性不一样

```apl
db.products.insert([
    {
      "product_name" : "Spy Coat",
      "product_attributes" : {
        "material" : [ "Tweed", "Wool", "Leather" ],
        "size" : {
          "length" : 72,
          "units" : "inches"
        }
      }
    },
    {
      "product_name" : "Spy Pen",
      "product_attributes" : {
         "colors" : [ "Blue", "Black" ],
         "secret_feature" : {
           "name" : "laser",
           "power" : "1000",
           "units" : "watts",
         }
      }
    },
    {
      "product_name" : "Spy Book"
    }
])
```

- 创建通配符索引

```apl
db.products.createIndex( { "product_attributes.$**" : 1 } )
```

- 测试，通配符索引可以支持任意单字段查询 product_attributes 或其嵌入字段

```apl
db.products.find( { "product_attributes.size.length" : { $gt : 60 } } )
db.products.find( { "product_attributes.material" : "Leather" } )
db.products.find( { "product_attributes.secret_feature.name" : "laser" } )
```

##### 注意事项

- 通配符索引不兼容的索引类型或属性

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/CC60857D24F54C718A9F0A09A1AE1A6F/40210)

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/486D8ED970CB43489C43A2283938C4B4/40650)

- 通配符索引是稀疏的，不索引空字段，因此，**通配符索引不能支持查询字段不存在的文档**
- 通配符索引不能支持以下查询：

```apl
db.products.find( {"product_attributes" : { $exists : false } } )
db.products.aggregate([
  { $match : { "product_attributes" : { $exists : false } } }
])
```

- 通配符索引为文档或数组的内容生成条目，而不是文档/数组本身
  - 因此，**通配符索引不能支持精确的文档/数组相等匹配**
  - 通配符索引可以支持查询字段等于空文档{}的情况

```shell
# 通配符索引不能支持以下查询:
db.products.find({ "product_attributes.colors" : [ "Blue", "Black" ] } )

db.products.aggregate([{
  $match : { "product_attributes.colors" : [ "Blue", "Black" ] } 
}])
```

### 2.6 索引类型

#### 唯一索引（Unique Indexes）

- 通过建立唯一性索引，可以保证集合中文档的指定字段拥有唯一值

```shell
# 创建唯一索引
db.values.createIndex({title:1},{unique:true})

# 复合索引支持唯一性约束
db.values.createIndex({title:1，type:1},{unique:true})

#多键索引支持唯一性约束
db.inventory.createIndex( { ratings: 1 },{unique:true} )
```

- **唯一性索引对于文档中缺失的字段，会使用null值代替**
  - 因此不允许存在多个文档缺失索引字段的情况
- **对于分片的集合，唯一性约束必须匹配分片规则**
  - 即：为了保证全局的唯一性，分片键必须作为唯一性索引的前缀字段

#### 部分索引（Partial Indexes）

- **部分索引仅对满足指定过滤器表达式的文档进行索引**
- 通过在一个集合中为文档的一个子集建立索引，**部分索引具有更低的存储需求和更低的索引创建和维护的性能成本，是3.2新版功能**
- 部分索引提供了稀疏索引功能的超集，应该优先于稀疏索引
- 注意：**如果同时指定了部分索引和唯一约束，那么唯一约束只适用于满足过滤表达式的文档**

```apl
db.restaurants.createIndex(
   { cuisine: 1, name: 1 },
   { partialFilterExpression: { rating: { $gt: 5 } } }
)
```

- partialFilterExpression：接收指定过滤条件的文档
  - 等式表达式(例如:field: value或使用$eq操作符)
  - [$exists: true](https://docs.mongodb.com/manual/reference/operator/query/exists/#mongodb-query-op.-exists)
  - $gt， $gte， $lt， $lte
  - $type 
  - 顶层的$and

```shell
# 符合条件，使用索引
db.restaurants.find( { cuisine: "Italian", rating: { $gte: 8 } } )
# 不符合条件，不能使用索引
db.restaurants.find( { cuisine: "Italian" } )
```

##### 案例1

- 测试数据

```json
db.restaurants.insert({
   "_id" : ObjectId("5641f6a7522545bc535b5dc9"),
   "address" : {
      "building" : "1007",
      "coord" : [
         -73.856077,
         40.848447
      ],
      "street" : "Morris Park Ave",
      "zipcode" : "10462"
   },
   "borough" : "Bronx",
   "cuisine" : "Bakery",
   "rating" : { "date" : ISODate("2014-03-03T00:00:00Z"),
                "grade" : "A",
                "score" : 2
              },
   "name" : "Morris Park Bake Shop",
   "restaurant_id" : "30075445"
})
```

- 创建索引

```json
db.restaurants.createIndex(
   { borough: 1, cuisine: 1 },
   { partialFilterExpression: { 'rating.grade': { $eq: "A" } } }
)
```

- 测试

```json
db.restaurants.find( { borough: "Bronx", 'rating.grade': "A" } )
db.restaurants.find( { borough: "Bronx", cuisine: "Bakery" } )
```

###### 唯一约束结合部分索引使用导致唯一约束失效的问题

- 注意：如果同时指定了 partialFilterExpression 和唯一约束，那么唯一约束只适用于满足筛选器表达式的文档。
  - 如果文档不满足筛选条件，那么带有唯一约束的部分索引不会阻止插入不满足唯一约束的文档。

##### 案例2

- 测试数据

```json
db.users.insertMany( [
   { username: "david", age: 29 },
   { username: "amanda", age: 35 },
   { username: "rajiv", age: 57 }
] )
```

- 创建索引，指定username字段和部分过滤器表达式age: {$gte:21} 的唯一约束

```json
db.users.createIndex(
   { username: 1 },
   { unique: true, partialFilterExpression: { age: { $gte: 21 } } }
)
```

- 测试

```json
// 索引防止了以下文档的插入，因为文档已经存在，且指定的用户名和年龄字段大于21
db.users.insertMany( [
   { username: "david", age: 27 },
   { username: "amanda", age: 25 },
   { username: "rajiv", age: 32 }
] )
```

```json
// 但是以下具有重复用户名的文档是允许的，因为唯一约束只适用于年龄大于或等于21岁的文档
db.users.insertMany( [
   { username: "david", age: 20 },
   { username: "amanda" },
   { username: "rajiv", age: null }
] )
```

#### 稀疏索引（Sparse Indexes）

- 索引的稀疏属性确保 **索引只包含具有索引字段的文档的条目，索引将跳过没有索引字段的文档**
  - 即：只对存在字段的文档进行索引（包括字段值为null的文档）
- **如果稀疏索引会导致查询和排序操作的结果集不完整，MongoDB将不会使用该索引**，除非 hint() 明确指定索引

```apl
# 只索引包含 xmpp_id 字段的文档
db.addresses.createIndex( { "xmpp_id": 1 }, { sparse: true } )
```

##### 案例1

- 数据准备

```json
db.scores.insertMany([
    {"userid" : "newbie"},
    {"userid" : "abby", "score" : 82},
    {"userid" : "nina", "score" : 90}
])
```

- 创建稀疏索引

```apl
db.scores.createIndex( { score: 1 } , { sparse: true } )
```

- 测试

```apl
# 使用稀疏索引
db.scores.find( { score: { $lt: 90 } } )

# 即使排序是通过索引字段，MongoDB也不会选择稀疏索引来完成查询，以返回完整的结果
db.scores.find().sort( { score: -1 } )

# 要使用稀疏索引，使用hint()显式指定索引
db.scores.find().sort( { score: -1 } ).hint( { score: 1 } )
```

##### 案例2

- 同时具有稀疏性和唯一性的索引 **可以防止集合中存在字段值重复的文档，但允许不包括此索引字段的文档插入**

```apl
# 创建具有唯一约束的稀疏索引
db.scores.createIndex( { score: 1 } , { sparse: true, unique: true } )
```

- 测试

```apl
# 索引允许以下文档插入
db.scores.insertMany( [
   { "userid": "AAAAAAA", "score": 43 },
   { "userid": "BBBBBBB", "score": 34 },
   { "userid": "CCCCCCC" },
   { "userid": "CCCCCCC" }
] )
```

```apl
# 索引不允许一下文档插入，因为score重复
db.scores.insertMany( [
   { "userid": "AAAAAAA", "score": 82 },
   { "userid": "BBBBBBB", "score": 90 }
] )
```

#### TTL索引（TTL Indexes）

- 一般系统中，通常会对过期且不再使用的数据进行老化处理
- 通常做法如下：
  1. 为每个数据记录一个时间戳，开启一个定时任务按时间戳定期删除过期数据
  2. 数据按日期进行分表，使用定时器删除过期的表
- Mongo提供了一种 TTL（Time To Live）索引
  - **需要声明在一个日期类型的字段中，是特殊的单字段索引**
  - 可以用它在一定时间或特定时间后，自动从集合中删除文档

```apl
# 创建 TTL 索引，TTL 值为3600秒
db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 3600 } )
```

- 对集合创建TTL索引后，MongoDB会在周期性运行的后台线程中，对该集合进行检查及数据清理工作。
  - 除了 **数据老化清除功能，TTL也具有普通索引功能**，同样可以用于加速数据检索
- TTL索引不保证过期数据会在过期后立即被删除
  - **删除过期文档的后台任务每 60s 运行一次**
  - 因此，在文档到期和后台任务运行之间的时间段内，文档仍然保留在集合中

##### 案例

- 数据准备

```apl
db.log_events.insertOne( {
   "createdAt": new Date(),
   "logEvent": 2,
   "logMessage": "Success!"
} )
```

- 创建TTL索引

```apl
db.log_events.createIndex( { "createdAt": 1 }, { expireAfterSeconds: 20 } )
```

###### **可变的过期时间**

- TTL索引在创建后，仍然可以对过期时间进行修改
- 使用 collMod 命令对索引的定义进行变更

```apl
db.runCommand({collMod:"log_events",index:{keyPattern:{createdAt:1},expireAfterSeconds:600}})
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/F4C4A9396F824D65B198946A2F87A3CF/40115)

##### 使用约束

- 使用TTL索引时，需注意以下的限制
  1. TTL索引只能支持单个字段，并且必须是非_id字段
  2. TTL索引不能用于固定集合
  3. TTL索引无法保证及时的数据老化清除（默认60s，db压力过大时可能更久）
  4. TTL索引对数据的清理仅仅使用了remove命令，并不高效，在TTL Monitor运行期间，可能对系统CPU、磁盘都会造成一定压力。相比之下，按日期分表的方式操作会更加高效。

#### 隐藏索引（Hidden Indexes）

- 隐藏索引对查询规划器不可见，不能用于支持查询。
- 通过隐藏索引，用户可以在不实际删除索引的情况下，评估删除索引后的影响，如果影响是负面的，用户可以取消隐藏索引，而不必重新创建索引。
- 4.4新版功能

##### 案例

```apl
# 创建隐藏索引
db.restaurants.createIndex({ borough: 1 },{ hidden: true });
# 隐藏现有索引
db.restaurants.hideIndex( { borough: 1} );
db.restaurants.hideIndex( "索引名称" )
# 取消隐藏索引
db.restaurants.unhideIndex( { borough: 1} );
db.restaurants.unhideIndex( "索引名称" ); 
```

- 准备数据

```apl
db.scores.insertMany([
    {"userid" : "newbie"},
    {"userid" : "abby", "score" : 82},
    {"userid" : "nina", "score" : 90}
])
```

- 创建隐藏索引

```apl
db.scores.createIndex(
   { userid: 1 },
   { hidden: true }
)
```

- 查看索引信息
  - 索引属性 hidden 只在值为 true 时返回

```apl
db.scores.getIndexes()
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/8266DB18517F4FD1B236F86118BB108C/40053)

- 测试

```apl
# 不使用索引
db.scores.find({userid:"abby"}).explain()
# 取消隐藏索引
db.scores.unhideIndex( { userid: 1} )
# 使用索引
db.scores.find({userid:"abby"}).explain()
```

### 2.7 索引使用建议

#### 1. 为每一个查询建立合适的索引

- 这是针对文档数较大（几十上百万）的集合，如果没有索引，每次Mongo都要把所有文档读到内存，对性能影响太大

#### 2. 创建合适的复合索引，不要依赖于交叉索引

- 如果查询要使用到多个字段，MongoDB有两种索引技术可以使用：
  - 交叉索引
    - 针对每个字段单独建立一个单字段索引，然后在查询执行时，使用相应的单字段索引进行索引交叉而得到查询结果
    - 交叉索引目前触发率较低，所以一般建议使用复合索引
  - 复合索引

```apl
# 查找所有年龄小于30岁的深圳市马拉松运动员
db.athelets.find({sport: "marathon", location: "sz", age: {$lt: 30}}})
# 创建复合索引
db.athelets.createIndex({sport:1, location:1, age:1})
```

#### 3. 复合索引字段排序：匹配条件在前，范围条件在后

- 原理大致同MySQL  B+Tree 设定，前面字段有序后面字段索引才好用得上

#### 4. 尽可能使用覆盖索引（Covered Index）

- 建议只返回需要的字段，同时利用覆盖索引来提升性能

#### 5. 建索引要在后台运行

- 在对一个集合创建索引时，该集合所在的数据库将不接受其他读写操作
- **对大数据量的集合建索引，建议使用后台运行选项 { background:true }**

#### 6. 避免设计过长的数组索引

- 数组索引是多值的，在存储时需要使用更多的空间，如果索引数组长度特别长，或者数组的增长不受控制，则可能导致索引空间急剧膨胀。

### 2.8 explain执行计划详解

- 通常关心：
  - 查询是否使用了索引
  - 索引是否减少了扫描的记录行数
  - 是否存在低效的内存排序
- MongoDB提供了 explain 命令，帮助评估指定查询模型的执行计划
- explain() 方法形式如下：
  - verbose：可选参数，表示执行计划的输出模式，默认 queryPlanner

| 模式名字          | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| queryPlanner      | 执行计划的详细信息，包括查询计划、集合信息、查询条件、最佳执行计划、查询方式和 MongoDB 服务信息等 |
| exectionStats     | 最佳执行计划的执行情况和被拒绝的计划等信息                   |
| allPlansExecution | 选择并执行最佳执行计划，并返回最佳执行计划和其他执行计划的执行情况 |

```apl
db.collection.find().explain(<verbose>)
```

#### queryPlanner

```apl
# 未创建title的索引
db.books.find({title:"book-1"}).explain("queryPlanner")
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/3A852A7610C343FF9C7C2B1AA5747B7A/40143)

| **字段名称**   | **描述**          |
| -------------- | ----------------- |
| plannerVersion | 执行计划的版本    |
| namespace      | 查询的集合        |
| indexFilterSet | 是否使用索引      |
| parsedQuery    | 查询条件          |
| winningPlan    | 最佳执行计划      |
| stage          | 查询方式          |
| filter         | 过滤条件          |
| direction      | 查询顺序          |
| rejectedPlans  | 拒绝的执行计划    |
| serverInfo     | mongodb服务器信息 |

#### executionStats

- executionStats 返回信息包含了 queryPlanner 的所有字段，并且还包含了最佳执行计划的执行情况

```apl
# 创建索引
db.books.createIndex({title:1})

db.books.find({title:"book-1"}).explain("executionStats")
```

![](https://note.youdao.com/yws/public/resource/0d777b87c34ec809163471a66bb5033c/xmlnote/68BFF24E673F43D0BEFADEA0D35FC6D2/40163)

| **字段名称**                                                 | **描述**                                             |
| ------------------------------------------------------------ | ---------------------------------------------------- |
| winningPlan.inputStage                                       | 用来描述子stage，并且为其父stage提供文档和索引关键字 |
| winningPlan.inputStage.stage                                 | 子查询方式                                           |
| winningPlan.inputStage.keyPattern                            | 所扫描的index内容                                    |
| winningPlan.inputStage.indexName                             | 索引名                                               |
| winningPlan.inputStage.isMultiKey                            | 是否是Multikey。如果索引建立在array上，将是true      |
| executionStats.executionSuccess                              | 是否执行成功                                         |
| executionStats.nReturned                                     | 返回的个数                                           |
| executionStats.executionTimeMillis                           | 这条语句执行时间                                     |
| executionStats.executionStages.executionTimeMillisEstimate   | 检索文档获取数据的时间                               |
| executionStats.executionStages.inputStage.executionTimeMillisEstimate | 扫描获取数据的时间                                   |
| executionStats.totalKeysExamined                             | 索引扫描次数                                         |
| executionStats.totalDocsExamined                             | 文档扫描次数                                         |
| executionStats.executionStages.isEOF                         | 是否到达 steam 结尾，1 或者 true 代表已到达结尾      |
| executionStats.executionStages.works                         | 工作单元数，一个查询会分解成小的工作单元             |
| executionStats.executionStages.advanced                      | 优先返回的结果数                                     |
| executionStats.executionStages.docsExamined                  | 文档检查数                                           |

#### allPlanExecution

- 包含 executionStats 内容，还有 allPlansExecution:[] 块

```apl
"allPlansExecution" : [
      {
         "nReturned" : <int>,
         "executionTimeMillisEstimate" : <int>,
         "totalKeysExamined" : <int>,
         "totalDocsExamined" :<int>,
         "executionStages" : {
            "stage" : <STAGEA>,
            "nReturned" : <int>,
            "executionTimeMillisEstimate" : <int>,
            ...
            }
         }
      },
      ...
   ]
```

##### stage状态

| **状态**        | **描述**                               |
| --------------- | -------------------------------------- |
| COLLSCAN        | 全表扫描                               |
| IXSCAN          | 索引扫描                               |
| FETCH           | 根据索引检索指定文档                   |
| SHARD_MERGE     | 将各个分片返回数据进行合并             |
| SORT            | 在内存中进行了排序                     |
| LIMIT           | 使用limit限制返回数                    |
| SKIP            | 使用skip进行跳过                       |
| IDHACK          | 对_id进行查询                          |
| SHARDING_FILTER | 通过mongos对分片数据进行查询           |
| COUNTSCAN       | count不使用Index进行count时的stage返回 |
| COUNT_SCAN      | count使用了Index进行count时的stage返回 |
| SUBPLA          | 未使用到索引的$or查询的stage返回       |
| TEXT            | 使用全文索引进行查询时候的stage返回    |
| PROJECTION      | 限定返回字段时候stage的返回            |

- 执行计划的返回结果中尽量不要出现以下stage:

  - COLLSCAN(全表扫描)

  - SORT(使用sort但是无index)

  - 不合理的SKIP

  - SUBPLA(未用到index的$or)

  - COUNTSCAN(不使用index进行count)
