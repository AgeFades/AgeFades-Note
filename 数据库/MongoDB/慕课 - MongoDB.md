# 慕课 - MongoDB

## MongoDB 入门

### 传统关系型数据库

![UTOOLS1577500101622.png](http://yanxuan.nosdn.127.net/91b82824f3fe8197cd02fb7330c44719.png)

### 常见NoSQL 数据库

![UTOOLS1577500172896.png](http://yanxuan.nosdn.127.net/15e59796080d57df66bde7d45c2d5615.png)

### MongoDB

```shell
# 存储文档的非关系型数据库，存储格式为类 JSON 形式的 BSON

# 同一个集合里可以放字段完全不同的数据，MongoDB 并不会制约我们的数据格式
```

![UTOOLS1577500246743.png](http://yanxuan.nosdn.127.net/12225396451001df045b19fc9a3d0ebd.png)

### Docker 运行 Mongo

```shell
# 请参考 运维目录下的 《Linux 基础环境搭建》

# 默认端口为 27017

# Mongo Express: Mongo 的 Web客户端，略...
```

### Mongo Shell

```shell
# mongo shell: 用来操作 MongoDB 的 JavaScript 脚本工具

# 运行 mongo shell 命令:
	# 第一个 mongo 是你的容器名字，第二个 mongo 指使用 mongo shell 与容器交互
docker exec -it mongo mongo
```

#### Mongo Shell 简单操作

```shell
# 重点: mongo shell 是一个 JavaScript 的脚本工具

# 简单的命令脚本:
print('hello'); 	# 输出语句

exit # 退出 mongo shell
```

#### Mongo Shell 基本操作

```shell
# 查看所有命令:
```

![UTOOLS1577520942811.png](http://yanxuan.nosdn.127.net/db52f4005691edf71470c012e9ecd066.png)

```shell
# 基本操作就是 CRUD

# MongoDB 的文档主键字段是 _id:
	# 文档主键的唯一性
	
	# 支持所有数据类型（数组除外）
	
	# 复合主键（利用另一篇文档当该篇文档的主键）
	
	# 默认 MongoDB 会生成对象主键 ObjectId:
		# 快速生成的 12 字节 id，包含创建时间，可排序，不能精确到秒级的创建时间
```

##### 创建文档

```shell
# 命令:
	# 支持单条数据插入，和多条数据插入
	# 返回的结果: 只有写入文档的数量
	# 错误时会返回报错信息以及报错文档，写入、修改、删除、匹配的条数...
db.collection.insert()


# 前置基础操作:
use test # 使用 test 数据库，返回 switched to db test

show collections # 查看 test 数据库中的集合

# 创建单个文档语法: db.collection.insertOne()
# writeConcern 定义了本次文档创建操作的安全写级别
	# 简单来说，安全写级别用来判断一次数据库写入操作是否成功
	# 安全写级别越高，丢失数据的风险就越低，然而写入操作的延迟也可能更高
	# 如果不提供 writeConcern 文档，mongoDB 使用默认的安全写级别
db.<collection>.insertOne(
	<document>,
	{
		writeConcern: <document>	
	}
)

# demo:
db.accounts.insertOne(
    {
        "_id": "account1",
        "name": "alice",
        "balance": 100
    }
) 
 
# 返回结果:
	# acknowledged: true 表示安全写级别被启用
		# 由于在 insertOne 命令中，没有提供 wrireConcern 文档，这里显示 mongo 默认的安全写级别启用状态
		
	# insertedId: 显示了被写入的文档的 _id
	
	# insertOne 命令会自动创建相应的集合
{
    "acknowledged" : true,
    "insertedId" : "account1"
}
```

```shell
# 命令:
db.collection.save()

# 与 insertOne() 作用基本相同
	# 如果不指定 _id 字段，save() 方法类似于 insert() 方法
	# 如果指定 _id 字段，则会更新该 _id 的数据
```

```shell
# 命令:
db.collection.insertMany()

# 作用: 插入多条

# demo:
db.accounts.insertMany(
    [{
        "_id": "account2",
        "name": "tony",
        "balance": 100
    },
    {
        "_id": "account3",
        "name": "jack",
        "balance": 300
    }]
) 

# 在插入多条文档顺序写入时，一旦遇到错误，操作就会退出，剩余的文档无论正确与否，都不会被写入
	# 而乱序写入时，及时某些文档造成了错误，剩余的正确文档仍然会被写入
```

##### 更新文档

```shell
# mongo 使用 update() 和 save() 方法来更新集合中的文档
```

```shell
# 语法格式:
	# query: update 的查询条件，类似 sql update 查询内 where 后面的
	# update: update 的对象和一些更新的操作符（如 $，$inc...）等，类似 sql update 中 set 后面的
	# upsert: 可选，表示不存在记录是否插入，true 为插入，默认为 false 不插入
	# multi: 可选，mongo 默认 false，只更新找到的第一条记录，true 时为全部更新所有符合条件的文档
	# writeConcern: 可选，抛出异常的级别。
db.collection.update(
	<query>,
	<update>,
	{
		upsert: <boolean>,
		multi: <boolean>,
		writeConcern: <documen>
	}
)

# demo:
db.accounts.update(
    {"name":"tony"},
    {$set:{
        "name":"tonyUpd"
        }
    },
    {
    	multi: true
    }
 )
```

```shell
# save() 方法上面已经讲过了
```

##### 删除文档

```shell
# mongo 中 remove() 函数用来移除集合中的数据
```

```shell
# 语法:
db.collection.deleteMany({})	# 删除所有数据

db.collection.deleteMany({"name":"tony"})	# 删除 name 为 tony 的全部文档

db.collection.deleteOne({"name":"tony"}) # 删除 name 为 tony 的一个文档

# delete 并不会真正释放空间
	# 需要执行 db.repairDatabase() 或者 db.runCommand({ repairDatabase:1 })
	# 这样才能回收磁盘空间
```

##### 查询文档

```shell
# mongo 查询文档使用 find() 函数
```

```shell
# 语法:
	# query: 可选，使用查询操作符指定查询条件
	# projection: 可选，使用投影操作符指定返回的键。
		# 查询时返回文档中所有的键值，只需省略该参数即可（默认）
db.collection.find(query, projection)

# 如果需要以 易读 的方式来读取数据，可以使用 pretty() 方法，语法格式如下:
	# 以格式化的方式来显示所有文档
db.collection.find().pretty()

# 除了 find() 还有 findOne()
```

##### Mongo 与 SQL 语句比较

| 操作       | 格式                     | 范例                                | SQL 类似语句        |
| ---------- | ------------------------ | ----------------------------------- | ------------------- |
| 等于       | { <key>:<value> }        | db.test.find( {"name":"tony"} )     | where name = 'tony' |
| 小于       | { <key>:{$lt:<value>} }  | db.test.find( { "age":{$lt:50} } )  | where age < 50      |
| 小于或等于 | { <key>:{$lte:<value>} } | db.test.find( { "age":{$lte:50} } ) | where age <= 50     |
| 大于       | {<key>:{$gt:<value>}}    | db.col.find({"likes":{$gt:50}})     | where likes > 50    |
| 大于或等于 | {<key>:{$gte:<value>}}   | db.col.find({"likes":{$gte:50}})    | where likes >= 50   |
| 不等于     | {<key>:{$ne:<value>}}    | db.col.find({"likes":{$ne:50}})     | where likes != 50   |

##### Mongo AND 条件

```shell
# 语法:
db.collection.find( {key1:value1, key2:value2} )

# demo:
db.accounts.find({"name":"tonyUpd","balance":100})

# 类似于 sql 中 AND
where name='tonyUpd' AND balance=100
```

##### Mongo OR 条件

```shell
# 语法:
db.collection.find(
	{
		$or: [
			{key1:value1}, {key2:value2}
		]
	}
)

# demo:
db.accounts.find(
    {
        $or: [
            {"name":"tonyUpd"},{"balance":100}
        ]
    }
)
```

##### Mongo 区间查询

```shell
# 语法:
db.collection.find(
	{
		balance: {$gt: 100},
		balance: {$lt: 300}
	}
)
```

##### Mongo 模糊查询

```shell
# 查询 name 中包含 x 的文档
db.collection.find( {name: /x/} )

  # demo:
  db.accounts.find( {name: /tony/} )

# 查询 name 中以 x 开头的文档
db.collection.find( {name: /^x/} )

  # demo:
  db.accounts.find(	{name: /^t/} )
  
# 查询 name 中以 x 结尾的文档
db.collection.find( {name: /x$/} )
	
	# demo:
	db.accounts.find( {name: /d$/} )
```

##### Mongo $type 操作符

```shell
# Mongo 使用的类型如下:
```

| **类型**                | **数字** | **备注**         |
| :---------------------- | :------- | :--------------- |
| Double                  | 1        |                  |
| String                  | 2        |                  |
| Object                  | 3        |                  |
| Array                   | 4        |                  |
| Binary data             | 5        |                  |
| Undefined               | 6        | 已废弃。         |
| Object id               | 7        |                  |
| Boolean                 | 8        |                  |
| Date                    | 9        |                  |
| Null                    | 10       |                  |
| Regular Expression      | 11       |                  |
| JavaScript              | 13       |                  |
| Symbol                  | 14       |                  |
| JavaScript (with scope) | 15       |                  |
| 32-bit integer          | 16       |                  |
| Timestamp               | 17       |                  |
| 64-bit integer          | 18       |                  |
| Min key                 | 255      | Query with `-1`. |
| Max key                 | 127      |                  |

```shell
# 如果想要获取集合中 name 字段为 String 类型的数据，可以使用如下命令:
db.collection.find( {"name": {$type:2}} ) 
# 或
db.collection.find( {"name": {$type:'string'}} )
```

##### Mongo 中 Limit 与 Skip

```shell
# Mongo 中使用 limit() 方法接受一个数字参数，指定读取记录条数

# 语法:
db.collection.find().limit(x)
	
# demo:
db.accounts.find().limit(2)
```

```shell
# Mongo 中使用 limit() 方法来跳过指定数量的数据

# 语法:
db.collction.find().limit(number).skip(number)

# demo:	从第二条开始查，查两条
db.accounts.find().limit(2).skip(1)
```

##### Mongo 中排序

```shell
# 在 Mongo 中使用 sort() 方法对数据进行排序，1 -> 升序, -1 -> 降序

# 语法:
db.collection.find().sort( {KEY:1} )

# demo:
db.accounts.find().sort( {balance:-1} )

# skip、limit、sort 三个放在一起执行的时候，执行顺序为 sort -> skip -> limit
```

