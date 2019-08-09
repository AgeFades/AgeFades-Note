# 黄康 - MongoDB 命令大全

## Mongo 简介

```shell
文档数据模型
BSON 数据结构
Database -> collection > document
```

## 命令

### 新增

```shell
db.<collection>.save({
    "_id":"110",
    "name":"快乐的一只小青蛙",
    "discription":"呱呱呱"
}) # 添加数据，collection 为所选集合，以下均用 test 举例

db.test.insertMany([{
    "name":"快乐的一只小鸡仔",
    "discription":"咯咯咯"
},{
    "name":"快乐的一只小旺财",
    "discription":"喵喵喵"
}]) # 批量添加

db.test.update({
    "name":"快乐的一只小青蛙"
},{$set:{"price":"9.99"}},{multi:true}) # 给 test 集合中 name 为 xxx 的 document 添加一条字段 price，值为 9.99，multi:true 表示所有符合条件的 document，默认只修改一行
```

### 查询

```shell
db.test.find("name":"快乐的一只小青蛙") # KV 键值对查询
db.getCollection('test').distinct("type") # 根据字段属性查询
db.test.find() # 查询 test 集合下所有数据
```

### 删除

```shell
db.test.deleteOne({
    "name":"快乐的一只小旺财"
}) # 条件删除单条

db.test.deleteMany({
    "name":"快乐的一只小鸡仔"
}) # 条件删除多条

db.test.update({},{
    $unset:{
        "price":""
    }
},{
    multi:true
}) # 删除字段，multi 为全部删除
```

### 修改

```shell
db.test.update({}, {$rename:{"price":"money"}}, false, true);  # 修改字段名
db.testas.update( 
	{ 
		"phonename" : "小米" 
	},{
   		 $set:{
   			 "phonename" : "华为"
   		 } 
   	},false,true)  # 修改字段值

```

## 聚合查询

| SQL 操作/函数 | mongodb聚合操作       |
| ------------- | --------------------- |
| where         | $match                |
| group by      | $group                |
| having        | $match                |
| select        | $project              |
| order by      | $sort                 |
| limit         | $limit                |
| sum()         | $sum                  |
| count()       | $sum                  |
| join          | $lookup （v3.2 新增） |

```shell
db.test.aggregate([
    {
        $match:{
            "name":"快乐的一只小青蛙" # 根据名字匹配
        }
    },{
        $group:{
            _id:"description", # 根据描述分组统计
            count:{
                $sum:1
            }
        }
    },{
        $sort:{
            price:1 # 根据价格排序 1 升序，2 降序
        }
    },{
        $project:{
            "count":1,"_id":0 # 查询结果不显示 id
            }
    },{
        $limit:10 # 取几条数据，配合 skip 能进行分页
    },{
        $push:"name" # 将需要的字段数据进行返回
    },{
        $first:"price" # 第一个
    },{
        $last:"price" # 最后一个
    }，{
        $sum:"price" # 总和 $avg 平均 $max 最大 $min 最小
    }
])
```

