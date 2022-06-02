[TOC]

# Fox - MongoDB快速实战与基本原理

## 一、课程简介

- 本专题讲解的MongoDB版本：`v4.4.x`
- [官方文档](https://www.mongodb.com/docs/v4.4/)
- [课程大纲](https://www.processon.com/view/link/61bef8c70791294d047ada3f#map)

## 二、MongoDB介绍

### 2.1 简介

- 一个文档数据库（以JSON为数据模型）
- 由C++语言编写，旨在 `为WEB应用提供可扩展的高性能数据存储解决方案`
- MongoDB是一个 `介于关系数据库和非关系数据库之间的产品`，数据格式是 `BSON`
  - 一种类似JSON的二进制形式的存储格式，简称 Binary JSON，和JSON一样支持内嵌文档对象和数组对象
  - 几乎可以实现类似关系数据库单表查询的绝大部分功能，而且`还支持对数据建立索引`
  - 原则上，Oracle和MySQL能做的事，MongoDB都能做（包括ACID事务）
- MongoDB是一个开源 OLTP 数据库，灵活的JSON格式非常适合敏捷式开发、高可用和水平扩展的大数据应用
  - `OLTP`：联机(在线)事务处理
  - `OLAP`：联机(在线)分析处理

### 2.2 版本变迁

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/301E6D03B85541418ECD3BB5407C2F59/35870)

### 2.3 MongoDB vs 关系型数据

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/56812C41F32049F18807092EC3EBFAA9/36232)

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/9B039CF2AE2A4411912833A7B01C2323/39384)

####  MongoDB 与 传统RDBMS 的差异

- `半结构化`：
  - 一个集合中，文档所拥有的的字段并不需要是相同的，而且也不需要对所用的字段进行声明。
    - 除了松散的表结构，文档还可以支持多级的嵌套、数组等灵活的数据类型，非常契合面向对象的编程模型。
- `弱关系`：
  - MongoDB没有外键的约束，也没有非常强大的表连接能力。类似的功能需要使用聚合管道技术来弥补。

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/80DB0A83AF1C45B2A86CA23D9771C9C7/36243)

## 三、MongoDB快速开始

### 3.1 Linux安装MongoDB

#### 环境准备

- Linux Centos7
- 安装MongoDB社区版

[MongoDB Community Server下载地址](https://www.mongodb.com/try/download/community)

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/00DB8A6CC439438E8AAAA690FCCE75D6/35872)

#### 命令下载

```shell
# 创建目录
mkdir mongo
# 切换目录
cd mongo
# 下载Mongo-4.4.9
wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.4.9.tgz
# 解压
tar -zxvf mongodb-linux-x86_64-rhel70-4.4.9.tgz
# 重命名
mv mongodb-linux-x86_64-rhel70-4.4.9 mongo-4.4.9
```

#### 启动Mongo

```shell
# 创建数据和日志目录
mkdir -p mongodb/data mongodb/log
# 带参数启动Mongo Server
mongo-4.4.9/bin/mongod --port 27017 --dbpath=/data/mongo/mongodb/data --logpath=/data/mongo/mongodb/log/mongodb.log --bind_ip=0.0.0.0 --fork
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652163791841.png)

[启动时遇到的问题及解决方案](https://zhuanlan.zhihu.com/p/446153443)

##### 启动参数介绍

- --dbpath：指定数据文件存放目录
- --logpath：指定日志文件，注意是指定日志文件不是目录
- --logappend：使用追加的方式记录日志
- --port：指定端口，默认为27017
- --bind_ip：默认只监 localhost 网卡
- --fork：后台启动
- --auth：开启认证模式

#### 添加环境变量

- 为了便于执行mongo命令

```shell
# 编辑环境变量文件
vim /etc/profile

# 在底部添加如下两行
export MONGODB_HOME=/data/mongo/mongo-4.4.9
export PATH=$PATH:$MONGODB_HOME/bin

# 保存退出后刷新环境变量
source /etc/profile
```

#### 关闭Mongo

```shell
mongod --port=27017 --dbpath=/data/mongo/mongodb/data --shutdown
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652164720470.png)

#### 利用配置文件启动服务

```shell
# 切换目录
cd mongo-4.4.9
# 创建配置文件目录
mkdir conf
# 编辑配置文件
vim conf/mongo.conf
```

```yaml
# 配置文件内容
systemLog:
  destination: file
  path: /data/mongo/mongodb/log/mongodb.log # log path
  logAppend: true
storage:
  dbPath: /data/mongo/mongodb/data # data directory
  engine: wiredTiger  #存储引擎
  journal:            #是否启用journal日志
    enabled: true
net:
  bindIp: 0.0.0.0
  port: 27017 # port
processManagement:
  fork: true
```

- 保存退出后，用配置文件方式启动

```shell
mongod -f conf/mongo.conf
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652164770594.png)

### 3.2 Mongo Shell使用

- mongo shell是MongoDB的交互式JavaScript Shell界面

```shell
# 进入 mongo shell 命令
# --port 指定端口，默认为 27017
# --host 指定连接的主机地址，默认为 127.0.0.1
mongo --port=27017
mongo localhost:27017
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652164928815.png)

#### Shell中关闭服务

```javascript
use admin
db.shutdownServer()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652165055787.png)

#### 查看JavaScript解释器的版本

```javascript
interpreterVersion()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652165574998.png)

#### mongo shell常用命令

| **命令**                        | **说明**                         |
| ------------------------------- | -------------------------------- |
| show dbs \| show databases      | 显示数据库列表                   |
| use  数据库名                   | 切换数据库，如果不存在创建数据库 |
| db.dropDatabase()               | 删除数据库                       |
| show collections \| show tables | 显示当前数据库的集合列表         |
| db.集合名.stats()               | 查看集合详情                     |
| db.集合名.drop()                | 删除集合                         |
| show users                      | 显示当前数据库的用户列表         |
| show roles                      | 显示当前数据库的角色列表         |
| show profile                    | 显示最近发生的操作               |
| load("xxx.js")                  | 执行一个JavaScript脚本文件       |
| exit  \|  quit()                | 退出当前shell                    |
| help                            | 查看mongodb支持哪些命令          |
| db.help()                       | 查询当前数据库支持的方法         |
| db.集合名.help()                | 显示集合的帮助信息               |
| db.version()                    | 查看数据库版本                   |

#### 数据库操作

```shell
# 查看所有库
show dbs
# 切换到指定库，不存在则创建
use test
# 删除当前数据库
db.dropDatabase()
```

#### 集合操作

```shell
# 查看所有集合
show collections
# 创建集合
db.createCollection("emp")
# 删除集合
db.emp.drop()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652166017420.png)

##### 创建集合语法

- 注意：当集合不存在时，向集合中插入文档也会自动创建集合

```shell
db.createCollection(name, options)
```

- options参数

| 字段   | 类型 | 描述                                                         |
| ------ | ---- | ------------------------------------------------------------ |
| capped | 布尔 | （可选）如果为true，则创建固定集合。固定集合是指有着固定大小的集合，当达到最大值时，它会自动覆盖最早的文档。 |
| size   | 数值 | （可选）为固定集合指定一个最大值（以字节计）。如果 capped 为 true，也需要指定该字段。 |
| max    | 数值 | （可选）指定固定集合中包含文档的最大数量。                   |

#### 安全认证

##### 创建管理员账号

```shell
# 设置管理员用户名密码需要切换到admin库
use admin
# 创建管理员
db.createUser({user:"agefades",pwd:"agefades",roles:["root"]})
# 查看当前数据库所有用户信息
show users
# 显示可设置权限
show roles
# 显示所有用户
db.system.users.find()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652166763180.png)

##### 常用权限

| 权限名               | 描述                                                         |
| -------------------- | ------------------------------------------------------------ |
| read                 | 允许用户读取指定数据库                                       |
| readWrite            | 允许用户读写指定数据库                                       |
| dbAdmin              | 允许用户在指定数据库中执行管理函数，如索引创建、删除，查看统计或访问system.profile |
| dbOwner              | 允许用户在指定数据库中执行任意操作，增、删、改、查等         |
| userAdmin            | 允许用户向system.users集合写入，可以在指定数据库里创建、删除和管理用户 |
| clusterAdmin         | 只在admin数据库中可用，赋予用户所有分片和复制集相关函数的管理权限 |
| readAnyDatabase      | 只在admin数据库中可用，赋予用户所有数据库的读权限            |
| readWriteAnyDatabase | 只在admin数据库中可用，赋予用户所有数据库的读写权限          |
| userAdminAnyDatabase | 只在admin数据库中可用，赋予用户所有数据库的userAdmin权限     |
| dbAdminAnyDatabase   | 只在admin数据库中可用，赋予用户所有数据库的dbAdmin权限       |
| root                 | 只在admin数据库中可用。超级账号，超级权限                    |

##### 重新赋予用户操作权限

```javascript
db.grantRolesToUser("agefades", [
  {role: "clusterAdmin", db: "admin"},
  {role: "userAdminAnyDatabase", db: "admin"},
  {role: "readWriteAnyDatabase", db: "admin"}
])
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652167091632.png)

##### 删除用户

```shell
# 指定删除
db.dropUser("agefades")
# 删除当前数据库所有用户
db.dropAllUser()
```

##### 用户认证

- 返回1表示认证成功

```javascript
use admin
db.auth("agefades", "agefades")
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652167334354.png)

##### 创建应用数据库用户

```javascript
use appdb
db.createUser({user:"appdb",pwd:"appdb",roles:["dbOwner"]})
```

##### 认证连接

- 默认情况下，Mongo不会启用鉴权，需指定以鉴权模式启动

```shell
mongod -f conf/mongo.conf --auth
```

- 启用鉴权之后，连接需要提供身份认证

```shell
mongo -u appdb -p appdb --authenticationDatabase=appdb
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652167651202.png)

### 3.3 Docker安装

#### compose yml文件

-  vim docker-compose-mongo.yaml

```yaml
version: '3.4'
services:
  mysql:
    container_name: mongo
    hostname: mongo
    image: mongo:4.4.10
    restart: always
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: "agefades"
      MONGO_INITDB_ROOT_PASSWORD: "agefades"
    command:
      --wiredTigerCacheSizeGB 1
```

- 默认情况下，mongo会将 wiredTigerCacheSizeGB 设置为与主机总内存成比例的值，而不考虑可能对容器施加的内存限制。
- MONGO_INITDB_ROOT_USERNAME 和 MONGO_INITDB_ROOT_PASSWORD 都存在，就会启用身份认证

#### 简单部署文件

- vim deploy.sh

```shell
docker-compose -f docker-compose-mongo.yaml down
docker-compose -f docker-compose-mongo.yaml up -d
```

- sh deploy.sh
  - 执行脚本，跑docker-compose即可

#### 进入容器

```shell
# 与容器交互，执行容器内 mongo 命令，带身份参数认证连接
docker exec -it mongo mongo -u agefades -p agefades
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652168393808.png)

### 3.4 Mongo相关工具

[官网GUI工具 - COMPASS](https://www.mongodb.com/zh-cn/products/compass)

- 能在不需要知道 mongo 查询语法的前提下，便利地分析和理解数据库模式，并帮助可视化构建查询

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/1120A1EEB42F41DEB9716DE0C2528F39/36389)

[GUI工具 - Robo 3T (免费)](https://robomongo.org/)

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/030C068D37A0474CAB980F915FC8AF81/36388)

[GUI工具 - Studio 3T (收费，试用30天)](https://studio3t.com/download/)

[MongoDB Database Tools](https://www.mongodb.com/try/download/database-tools)

| 文件名称     | 作用               |
| ------------ | ------------------ |
| mongostat    | 数据库性能监控工具 |
| mongotop     | 热点表监控工具     |
| mongodump    | 数据库逻辑备份工具 |
| mongorestore | 数据库逻辑恢复工具 |
| mongoexport  | 数据导出工具       |
| mongoimport  | 数据导入工具       |
| bsondump     | BSON格式转换工具   |
| mongofiles   | GridFS文件工具     |

## 四、MongoDB文档操作

### 4.1 插入文档

- 3.2 版本之后新增了：
  - db.collection.insertOne()
  - db.collection.insertMany()

#### 新增单个文档

- insertOne 支持 `writeConcern`
  - `writeConcern` 决定一个写操作落到多少个节点上才算成功。
  - `writeConcern` 的取值包括：
    - 0：发起写操作，不关心是否成功
    - 1 ~ 集群最大数据节点数：写操作需要被复制到指定节点数才算成功
    - majority：写操作需要被复制到大多数节点上才算成功
- insert：若插入的数据主键已经存在，则会抛 DuplicateKeyException 异常，提示主键重复，不保存当前数据
- save：如果 _id 主键存在则更新数据，如果不存在就插入数据

```javascript
db.collection.insertOne(
	<document>,
  {
  	writeConcern: <document>
  }
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652169346120.png)

#### 批量新增文档

- insertMany：向指定集合中插入多条文档数据
  - writeConcern：写入策略，默认为1，即要求确认写操作，0是不要求
  - ordered：指定是否按顺序写入，默认 true，按顺序写入

```javascript
db.collection.insertMany(
	[ <document1>, <document2> ... ]
  {
  	writeConcern: <document>,
  	ordered: <boolean>
  }
)
```

- insert 和 save 也可以实现批量插入

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652169650047.png)

#### 测试执行JS脚本批量新增数据

- 这里使用Linux宿主机上的 mongo 服务
- vim book.js

```javascript
var tags = ["nosql","mongodb","document","developer","popular"];
var types = ["technology","sociality","travel","novel","literature"];
var books=[];
for(var i=0;i<50;i++){
    var typeIdx = Math.floor(Math.random()*types.length);
    var tagIdx = Math.floor(Math.random()*tags.length);
    var favCount = Math.floor(Math.random()*100);
    var book = {
        title: "book-"+i,
        type: types[typeIdx],
        tag: tags[tagIdx],
        favCount: favCount,
        author: "xxx"+i
    };
    books.push(book)
}
db.books.insertMany(books);
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652169882731.png)

- 进入 mongo shell，执行

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652169951574.png)

### 4.2 查询文档

- find 查询集合中的若干文档。语法格式如下：

```shell
db.collection.find(query, projection)
```

- `query`：
  - 可选，使用查询操作符指定查询条件
- `projection`：
  - 可选，使用投影操作符指定返回的键。
  - 查询时返回文档中所有键值，只需省略该参数即可（默认省略）
  - 投影时，_id为1的时候，其他字段必须是1；_id是0的时候，其他字段可以是0；如果没有_id字段约束，多个其他字段必须同为0或同为1。

#### 查询集合所有文档

```javascript
db.books.find()
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652170163844.png)

- 如果查询返回的条目数量较多，mongo shell 会自动实现分批显示。
  - 默认情况每次显示20条，可以输入 it 命令读取下一批。

#### 查询集合单个文档

- findOne 查询集合中的第一个文档。语法格式如下：

```javascript
db.collection.findOne(query, projection)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652170450446.png)

#### 条件查询

##### 指定条件查询

- 语法格式如下：

```shell
db.collection().find()

# 可以使用 pretty() 方法以格式化的方式来显示文档
db.collection.find().pretty()
```

- 示例

```shell
# 查询带有nosql标签的book文档
db.books.find({tag:"nosql"})
# 按照id查询单个book文档
db.books.find({_id:ObjectId("627a1c495e8b7862cb266ff6")})
# 查询分类为travel、收藏数超过60个的book文档
db.books.find({type:"travel",favCount:{$gt:60}})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652171114803.png)

##### 查询条件对照表

| SQL    | MQL            |
| ------ | -------------- |
| a = 1  | {a: 1}         |
| a <> 1 | {a: {$ne: 1}}  |
| a > 1  | {a: {$gt: 1}}  |
| a >= 1 | {a: {$gte: 1}} |
| a < 1  | {a: {$lt: 1}}  |
| a <= 1 | {a: {$lte: 1}} |

##### 查询逻辑对照表

| SQL             | MQL                                    |
| --------------- | -------------------------------------- |
| a = 1 AND b = 1 | {a: 1, b: 1}或{$and: [{a: 1}, {b: 1}]} |
| a = 1 OR b = 1  | {$or: [{a: 1}, {b: 1}]}                |
| a IS NULL       | {a: {$exists: false}}                  |
| a IN (1, 2, 3)  | {a: {$in: [1, 2, 3]}}                  |

##### 查询逻辑运算符

- $lt: 存在并小于
- $lte: 存在并小于等于
- $gt: 存在并大于
- $gte: 存在并大于等于
- $ne: 不存在或存在但不等于
- $in: 存在并在指定数组中
- $nin: 不存在或不在指定数组中
- $or: 匹配两个或多个条件中的一个
- $and: 匹配全部条件

```shell
db.books.find({$and: [{tag:"nosql"},{favCount:{$gte:80}}]})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652171316240.png)

#### 排序&分页

##### 指定排序

- Mongo中使用 sort() 方法对数据进行排序
  - 1 为升序排列、-1 为降序排列

```shell
# 示例：指定按收藏数 favCount 降序返回
db.books.find({type:"travel"}).sort({favCount:-1})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652171716875.png)

##### 分页查询

- skip 用于指定跳过记录数，limit 用于限定返回结果数量。

```shell
# 示例：假定每页大小4条，查询第3页的book文档
db.books.find().skip(8).limit(4)
```

###### 处理分页问题 - 巧分页

- 数据量大的时候，应该避免使用 skip + limit 形式的分页
- 替代方案：使用查询条件 + 唯一排序条件

```shell
# 示例如下：

# 第一页：
db.books.find().sort({_id:1}).limit(20)

# 第二页:
db.books.find({_id: {$gt: <第一页最后一个_id>}}).sort({_id:1}).limit(20)

# 第三页
db.books.find({_id: {$gt: <第二页最后一个_id>}}).sort({_id:1}).limit(20)
```

###### 处理分页问题 - 避免使用count

- 尽可能不要计算总页数，特别是数据量大和查询条件不能完整命中索引时。
- 考虑如下场景：假设集合总共有1000w条数据，在没有索引的情况下

```shell
# 只需要遍历前n条，直到找到50条 x=100 的文档即可结束
db.coll.find({x: 100}).limit(50);
# 需遍历完 1000w 条，找到所有符合要求的文档才能得到结果
# 为了计算总页数而进行的 count() 往往是拖慢页面整体加载速度的原因
db.coll.count({x: 100}); 
```

##### 正则表达式匹配查询

- mongo 使用 $regex 操作符来设置匹配字符串的正则表达式

```shell
# 使用正则表达式查找type包含 so 字符串的book
db.books.find({type: {$regex: "so"}})

# 或者
db.books.find({type: /so/})
```

### 4.3 更新文档

- 可用 update 命令对指定的数据进行更新，命令的格式如下：

```shell
db.collection.update(query, update, options)
```

- query：描述更新的查询条件
- update：描述更新的动作及新的内容
- options：描述更新的选项
  - upsert：可选，如果不存在update的记录，是否插入新的记录。默认false，不插入
  - multi：可选，是否按条件查询出的多条记录全部更新。默认false，只更新找到的第一条记录
  - writeConcern：可选，决定一个写操作落到多少个节点上才算成功

#### 更新操作符

| **操作符** | **格式**                                        | **描述**                                       |
| ---------- | ----------------------------------------------- | ---------------------------------------------- |
| $set       | {$set:{field:value}}                            | 指定一个键并更新值，若键不存在则创建           |
| $unset     | {$unset : {field : 1 }}                         | 删除一个键                                     |
| $inc       | {$inc : {field : value } }                      | 对数值类型进行增减                             |
| $rename    | {$rename : {old_field_name : new_field_name } } | 修改字段名称                                   |
| $push      | { $push : {field : value } }                    | 将数值追加到数组中，若数组不存在则会进行初始化 |
| $pushAll   | {$pushAll : {field : value_array }}             | 追加多个值到一个数组字段内                     |
| $pull      | {$pull : {field : _value } }                    | 从数组中删除指定的元素                         |
| $addToSet  | {$addToSet : {field : value } }                 | 添加元素到数组中，具有排重功能                 |
| $pop       | {$pop : {field : 1 }}                           | 删除数组的第一个或最后一个元素                 |

#### 更新单个文档

- 示例：某个book文档被收藏了，则需将该文档的 favCount 字段自增

```javascript
db.books.update(
	{_id: ObjectId("627a1c495e8b7862cb266fd5")},
  {$inc: {favCount:1}}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652250838068.png)

#### 更新多个文档

- 示例：将分类为 novel 的文档增加发布时间（publishedDate）

```javascript
db.books.update(
	{type:"novel"},
  {$set:{publishedDate:new Date()}},
  {"multi":true}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652250994658.png)

- update命令的选项配置较多，为了简化使用还可以使用一些快捷命令
  - updateOne：更新单个文档
  - updateMany：更新多个文档
  - replaceOne：替换单个文档

#### upsert命令

- upsert 是一种特殊的更新，其表现为如果目标文档不存在，则执行插入命令

```javascript
db.books.update(
	{title:"my book"},
  {$set: {tags:["nosql","mongodb"], type:"none", author:"agefades"}},
  {upsert:true}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652251212715.png)

- nMatched、nModified都为0，表示没有文档被匹配及更新
  - nUpserted=1提示执行了upsert动作

#### 实现replace语义

- update中的更新描述，通常由操作符描述，如果更新操作中不包含任何操作描述符，则会实现文档的 replace 语义

```javascript
db.books.update(
	{title: "my book"},
  {justTitle: "my first book"}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652251429032.png)

#### findAndModify命令

- findAndModify 兼容了查询和修改指定文档的功能，只能更新单个文档

```javascript
db.books.findAndModify(
	{
    query: {_id: ObjectId("627a1c495e8b7862cb266fd5")},
    update: {$inc: {favCount:1}}
  }
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652251560590.png)

- 该操作会返回符合查询条件的文档数据，并完成对文档的修改
  - 默认情况下，返回的是旧数据，如果希望返回修改后的新数据，则可以指定 new 选项

```javascript
db.books.findAndModify(
	{
    query: {_id: ObjectId("627a1c495e8b7862cb266fd5")},
    update: {$inc: {favCount:1}},
    new: true
  }
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652251650219.png)

- 与 findAndModify 语义相近的命令如下：
  - findOneAndUpdate：更新单个文档并返回文档
  - findOneAndReplace：替换单个文档并返回文档

### 4.4 删除文档

#### 使用remove删除文档

- remove 命令需配合查询条件使用
- 匹配查询条件的文档会被删除
- 指定一个空文档条件会删除所有文档

```javascript
// 删除 age=28 的记录
db.user.remove({age:28})
// 删除 age<25 的记录
db.user.remove({age:{$lt:25}})
// 删除所有记录
db.user.remove({})
// 报错
db.user.remove()
```

- remove命令会删除匹配条件的全部文档，如果希望明确限定只删除一个文档，则需要指定 justOne 参数，命令格式如下：

```javascript
db.collection.remove(query, justOne)

// 举例如下：
db.books.remove({type:"novel"},true)
```

#### 使用delete删除文档

- 官方推荐使用 deleteOne() 和 deleteMany() 删除文档，语法格式如下：

```javascript
// 删除集合下全部文档
db.books.deleteMany({})
// 删除 type=novel 的全部文档
db.books.deleteMany({type:"novel"})
// 删除 type=novel 的一个文档
db.books.deleteOne({type:"novel"})
```

- remove、deleteMany 等命令需要对查询范围内的文档逐个删除，如果希望删除整个集合，则直接使用 drop 命令会更高效

#### 返回被删除文档

- remove、deleteOne 等命令在删除文档后只会返回确认性的信息，如果希望获得被删除的文档，则可以使用 findOneAndDelete 命令

```javascript
// 示例：
db.books.findOneAndDelete({type:"novel"})
```

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/5853E5547551467C9D809E918BC2C772/36852)

- 除了在结果中返回删除文档，findOneAndDelete命令还被允许定义 "删除的顺序"，即按照指定顺序删除找到的第一个文档

```javascript
// 示例
db.books.findOneAndDelete({type:"novel"},{sort:{favCount:1}})
```

- remove、deleteOne 等命令只能按默认顺序删除，利用这个特性，findOneAndDelete 可以实现队列的先进先出

### 4.5 文档操作最佳实践

#### 关于文档结构

- 防止使用太长的字段名（浪费空间）
- 防止使用太深的数组嵌套（超过2层操作就比较复杂）
- 不使用中文、标点符号等非拉丁文字母作为字段名

#### 关于写操作

- update语句里只包括需要更新的字段
- 尽可能使用批量插入来提升写入性能
- 使用 TTL 自动过期 存日志类型的数据

## 五、MongoDB数据模型

### 5.1 BSON协议与数据类型

#### JSON

- JSON是当今非常通用的一种跨语言Web数据交互格式，属于 ECMAScript 标准规范的一个子集
- JSON只定义了6种数据类型：
  - string：字符串
  - number：数值
  - object：JS的对象形式，用 {kev:value} 表示，可嵌套
  - array：数组，JS的表达方式[value]，可嵌套
  - true/false：布尔类型
  - null：空值
- 大多数情况下，使用JSON作为数据交互格式已经是理想的选择
  - 但是 JSON 基于文本的解析效率并不是最好的，在某些场景往往会考虑更合适的编解码格式
  - 一些做法如：
    - 在微服务架构中，使用 gRPC（基于Google的Protobuf）可以获得更好的网络利用率
    - 分布式中间件、数据库，使用私有定制的TCP数据包格式来提高性能、低延时的计算能力

#### BSON

- BSON由10gen团队设计并开源，目前主要用于MongoDB数据库。
- BSON（Binary JSON）是二进制版本的JSON，在性能方面有更优的表现
- BSON 与 JSON 最大区别：
  - JSON 是基于文本的
  - BSON 是二进制（字节流）编解码的形式
  - 在空间的使用上，BSON相比JSON并没有明显的优势
- MongoDB在文档存储、命令协议上都采用了BSON作为 编解码 格式，主要具有如下优势：
  - 类JSON的轻量级语义，支持简单清晰的嵌套、数组层次结构，可以实现模式灵活的文档结构
  - 更高效的遍历，BSON在编码时会记录每个元素的长度，可以直接通过 seek 操作进行元素的内容读取，相对JSON解析来说，遍历速度更快
  - 更丰富的数据类型，除了JSON的基本数据类型，BSON还提供了MongoDB所需的一些扩展类型，比如日期、二进制数据等，这更加方便数据的表示和操作

##### BSON的数据类型

- MongoDB中，一个BSON文档最大大小为16M，文档嵌套的级别不超过100

[官方BSON数据类型文档](https://docs.mongodb.com/v4.4/reference/bson-types/)

| Type                       | Number | Alias                 | Notes                      |
| -------------------------- | ------ | --------------------- | -------------------------- |
| Double                     | 1      | "double"              |                            |
| String                     | 2      | "string"              |                            |
| Object                     | 3      | "object"              |                            |
| Array                      | 4      | "array"               |                            |
| Binary data                | 5      | "binData"             | 二进制数据                 |
| Undefined                  | 6      | "undefined"           | Deprecated.                |
| ObjectId                   | 7      | "objectId"            | 对象ID，用于创建文档ID     |
| Boolean                    | 8      | "bool"                |                            |
| Date                       | 9      | "date"                |                            |
| Null                       | 10     | "null"                |                            |
| Regular Expression         | 11     | "regex"               | 正则表达式                 |
| DBPointer                  | 12     | "dbPointer"           | Deprecated.                |
| JavaScript                 | 13     | "javascript"          |                            |
| Symbol                     | 14     | "symbol"              | Deprecated.                |
| JavaScript code with scope | 15     | "javascriptWithScope" | Deprecated in MongoDB 4.4. |
| 32-bit integer             | 16     | "int"                 |                            |
| Timestamp                  | 17     | "timestamp"           |                            |
| 64-bit integer             | 18     | "long"                |                            |
| Decimal128                 | 19     | "decimal"             | New in version 3.4.        |
| Min key                    | -1     | "minKey"              | 表示一个最小值             |
| Max key                    | 127    | "maxKey"              | 表示一个最大值             |

##### $type操作符

- $type操作符基于BSON类型来检索集合中匹配的数据类型，并返回结果

```javascript
// 示例如下：
db.books.find({"title":{$type:2}})
// 或者
db.books.find({"title":{$type:"string"}})
```

### 5.2 日期类型

- MongoDB的日期类型使用UTC进行存储，也就是+0时区的时间

```javascript
// 示例：
db.dates.insert(
	[
    {data1:Date()},
    {data2:new Date()},
    {data3:ISODate()}
  ]
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652257413690.png)

- 使用 new Date() 与 ISODate() 最终都会生成 ISODate 类型的字段（对应于UTC时间）

### 5.3 ObjectId生成器

- MongoDB集合中所有的文档都有一个唯一的_id字段，作为集合的主键
  - 默认情况下，_id字段使用ObjectId类型，采用16进制编码形式，共12个字节

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/E05541A33A9B4CADB0CFAE94CEA91E29/36980)

- 为了避免文档的_id字段出现重复，ObjectId被定义为3个部分：
  - 4字节表示Unix时间戳（秒）
  - 5字节表示随机数（机器号+进程号唯一）
  - 3字节表示计数器（初始化时随机）

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/F370CC17C330484691D3D7DB65B1D9D2/36989)

| 属性/方法               | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| str                     | 返回对象的十六进制字符串表示。                               |
| ObjectId.getTimestamp() | 将对象的时间戳部分作为日期返回。                             |
| ObjectId.toString()     | 以字符串文字“”的形式返回 JavaScript 表示ObjectId(...)。      |
| ObjectId.valueOf()      | 将对象的表示形式返回为十六进制字符串。返回的字符串是str属性。 |

- 生成一个新的ObjectId

```javascript
x = ObjectId()
```

### 5.4 内嵌文档和数组

#### 内嵌文档

- 示例如下

```javascript
db.books.insert({
  title: "撒哈拉的故事",
  author: {
    name: "三毛",
    gender: "女",
    hometown: "重庆"
  }
})
```

```javascript
// 查询三毛的作品
db.books.find({"author.name":"三毛"})
```

```javascript
// 修改三毛的故乡所在地
db.books.findAndModify(
  {
		query: {"author.name":"三毛"},
 	  update: {$set: {"author.hometown": "重庆/台湾"}},
  	new: true
	}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652258147706.png)

#### 数组

- 示例如下

```javascript
db.books.findAndModify(
	{
    query: {"author.name":"三毛"},
    update: {$set: {
      tags: ["旅行","随笔","散文"]
    }},
    new: true
  }
)
```

```javascript
// 查询数组元素
db.books.find({"author.name":"三毛"},{title:1,tags:1})
// 利用$slice获取最后一个tag
db.books.find({"author.name":"三毛"},{title:1,tags:{$slice:-1}})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652258389017.png)

- $slice 是一个查询操作符，用于指定数组的切片方式
- 数组末尾追加元素，可以使用 $push 操作符

```javascript
db.books.updateOne(
	{"author.name":"三毛"},
  {$push:{tags:"猎奇"}}
)

// $push可以配合其他操作符

// 比如和 $each 配合可用于添加多个元素
db.books.updateOne(
	{"author.name":"三毛"},
  {$push:{tags: {$each: ["伤感","想象力"]} }}
)

// 如果加上$slice操作符，那么只会保留经过切片后的元素
db.books.updateOne(
	{"author.name":"三毛"},
  {$push:{tags: {$each: ["伤感","想象力"], $slice: -3} }}
)
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652258709432.png)

- 根据元素查询

```javascript
// 会查出所有包含伤感的文档
db.books.find({tags:"伤感"})
// 会查出所有同时包含 伤感、想象力 的文档
db.books.find({tags:{$all:["伤感","想象力"]}})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652258815254.png)

#### 嵌套型的数组

- 数组元素可以是基本类型，也可以是内嵌的文档结构

```javascript
{
    tags:[
        {tagKey:xxx,tagValue:xxxx},
        {tagKey:xxx,tagValue:xxxx}
    ]
}
```

- 这种结构非常灵活，一个很适合的场景就是商品的多属性表示

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/7CB251ED86334CCE99A02D93461664FE/37235)

- 一个商品可以同时包含多个维度的属性，比如尺码、颜色、风格等，使用文档可表示为：

```javascript
db.goods.insertMany([{
    name:"羽绒服",
    tags:[
        {tagKey:"size",tagValue:["M","L","XL","XXL","XXXL"]},
        {tagKey:"color",tagValue:["黑色","宝蓝"]},
        {tagKey:"style",tagValue:"韩风"}
    ]
},{
    name:"羊毛衫",
    tags:[
        {tagKey:"size",tagValue:["L","XL","XXL"]},
        {tagKey:"color",tagValue:["蓝色","杏色"]},
        {tagKey:"style",tagValue:"韩风"}
    ]
}])
```

- 以上设计是一种常见的多值属性的做法，当我们需要根据属性进行检索时，需要用到 $elemMatch 操作符

```javascript
// 筛选出color=黑色的商品信息
db.goods.find({
    tags:{
        $elemMatch:{tagKey:"color",tagValue:"黑色"}
    }
})
```

- 如果进行组合式的条件检索，则可以使用多个 $elemMatch 操作符

```javascript
// 筛选出color=蓝色，并且size=XL的商品信息
db.goods.find({
    tags:{
        $all:[
            {$elemMatch:{tagKey:"color",tagValue:"黑色"}},
            {$elemMatch:{tagKey:"size",tagValue:"XL"}}
        ]  
    }
})
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652260292862.png)

### 5.5 固定集合

- 固定集合（capped collection）是一种限定大小的集合
  - capped 是覆盖、限额的意思。
  - 跟普通集合相比，数据在写入这种集合时遵循 FIFO 原则。
- 新文档在写入时，会被插入队列的末尾，如果队列已满，那么之前的文档就会被新写入的文档所覆盖。
  - 通过固定集合的大小，我们可以保证 **数据库只会存储"限额"的数据，超过该限额的旧数据都会被丢弃**

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/E7952C6A7010453D9C373287046CF4F0/37009)

#### 使用示例

- 创建固定集合
  - max：指集合的文档数量最大值，这里是10条
  - size：指集合的空间占用最大值，这里是4096字节（4kb）

```javascript
db.createCollection("logs",{capped:true,size:4096,max:10})
```

- 这两个参数会同时对集合的上限产生影响。也就是说，只要任一条件打到阈值都会认为集合已经写满。
  - 其中 size 是必选的，而 max 是可选的
- 可以使用 collection.stats 命令查看文档的占用空间

```javascript
// 示例
db.logs.stats()
```

- 测试：
  - 尝试在该集合中插入15条数据，再查询会发现，
  - 由于文档数量上限为10条，前面插入的5条已经被覆盖了

```javascript
for (var i = 0; i < 15; i++) {
  db.logs.insert(
  	{t: "row - " + i}
  )
}
```

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/08B768C5BC894EBE98CC395D31F434C1/37034)

#### 优势与限制

- **固定集合在底层使用的是顺序I/O操作**，而普通集合使用的是随机I/O
  - 顺序I/O在磁盘操作上由于寻道次数少而比随机I/O要高效得多
  - 因此，固定集合的写入性能是很高的。
  - 此外，如果按写入顺序进行数据读取，也会获得非常好的性能表现
- 但它也存在一些限制，主要有5个方面：
  1. 无法动态修改存储的上限
     - 如果需要修改max或size，则只能先删掉集合再重新创建
  2. 无法删除已有的数据
     - 对固定集合的数据进行删除会报错
     - ![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/205AFCBDFCD14D6F840215EDBD607553/37054)
  3. 对已有数据进行修改，新文档大小必须与原来的文档大小一致，否则不允许更新
     - ![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/976C990C0E104E79A592FD5AEBDE6B4C/37056)
  4. 默认情况下，固定集合只有一个_id索引，而且最好是按数据写入的顺序进行读取。
     - 也可以添加新的索引，但会降低数据写入的性能
  5. 固定集合不支持分片，在MongoDB 4.2版本规定了，事务中也无法对固定集合执行写操作

#### 适用场景

- 固定集合适合用来存储一些 "临时态" 的数据。
  - 临时态 意味着数据在一定程度上可以被丢弃，同时，用户更关注最新数据
  - 随着时间推移，数据重要性逐渐降低，直至被淘汰处理
- 一些适用的场景如下：
  - 系统日志
    - 这非常符合固定集合的特征，而日志系统通常也只需要一个固定的空间来存放日志。
    - 在MongoDB内部，副本集的同步日志（oplog）就使用了固定集合
  - 存储少量文档
    - 如：最新发布的TopN条文章信息。
    - 得益于内部缓存的作用，对于这种少量文档的查询是非常高效的
- 可以用来作消息队列

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652264158593.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652264174311.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1652264193685.png)

## 六、WriedTiger读写模型详解

### 6.1 WiredTiger介绍

- MongoDB从3.0开始，引入可插拔存储引擎的概念。
  - 目前主要有 MMAPV1、WriedTiger 存储引擎可供选择。
  - 在3.22源的消耗，节省约60%以上的硬盘资源

### 6.2 WiredTiger读写模型

#### 读缓存

- **理想情况下，MongoDB可以提供近似内存式的读写性能**
- WiredTiger引擎实现了数据的二级缓存
  - 第一层是操作系统的页面缓存
  - 第二层则是引擎提供的内部缓存

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/585EC221E6F74BA98435B7B5640DF97A/43548)

- 读取数据时的流程如下：
  1. 数据库发起Buffer I/O读操作，由操作系统将磁盘数据页加载到文件系统的页缓存区
  2. 引擎层读取页缓存区的数据，进行解压后存放到内部缓存区
  3. 在内存中完成匹配查询，将结果返回给应用
- **MongoDB为了尽可能保证业务查询的"热数据"能快速被访问，其内部缓存的默认大小达到了内存的一半**
  - 该值由 wiredTigerCacheSize 参数指定，其默认的计算公式如下：

```shell
wiredTigerCacheSize=Math.max(0.5*(RAM-1GB),256MB)
```

#### 写缓存

- 当数据发生写入时，MongoDB并不会立即持久化到磁盘上，
  - 而是先在内存中记录这些变更，之后通过 CheckPoint 机制将变化的数据写入磁盘。
- 这里处理主要有两个原因：
  - 如果每次写入都触发一次磁盘I/O，那么开销太大，而且响应延时会比较大
  - 多个变更的写入可以尽可能进行I/O合并，降低资源负荷

#### MongoDB单机下保证数据可靠性

- 主要包括以下两个部分

##### CheckPoint（检查点）机制

- 快照（snapshot）描述了某一时刻（point-in-time）数据在内存中的一致性视图，
  - 而这种数据的一致性是WiredTiger通过MVCC（多版本并发控制）实现的。
- 当建立CheckPoint时，WiredTiger会在内存中建立所有数据的一致性快照，
  - 并将该快照覆盖的所有数据变化一并进行持久化（fsync）
  - 成功之后，内存中数据的修改才得以真正保存
- **默认情况下，MongoDB每60秒建立一次CheckPoint**，
  - 在检查点写入过程中，上一个检查点仍然是可用的。
  - 这样可以保证一旦出错，MongoDB仍然能恢复到上一个检查点。
- CheckPoint的刷新周期可以调整 **storage.syncPeriodSecs参数（默认值60s）**
  - 在MongoDB 3.4及以下版本中，当Journal日志达到2GB时，同样会触发CheckPoint行为
  - 如果应用存在大量随机写入，则CheckPoint可能会造成磁盘I/O的抖动
  - 在磁盘性能不足的情况下，问题会更加显著，此时适当缩短CheckPoint周期可以让写入更平滑



##### Journal日志

- **Journal是一种预写式日志（write ahead log）机制**，主要用于弥补CheckPoint机制的不足。
- 如果开启了 Journal日志，那么WiredTiger会将每个写操作的redo日志写入Journal缓冲区，
  - 该缓冲区会频繁地将日志持久化到磁盘上。
  - **默认情况下，Journal缓冲区每100ms执行一次持久化**
    - 可以通过参数 **storage.journal.commitIntervalMs** 指定
    - MongoDB 3.4及以下版本默认是50ms，3.6之后调整到了100ms
    - 由于Journal日志采用的是循序I/O写操作，频繁地写入对磁盘的影响并不是很大。
  - 此外，Journal日志达到100MB，或是应用程序指定 journal:true，写操作都会触发日志的持久化
- 一旦MongoDB发生宕机，重启程序时会先恢复到上一个检查点，然后根据Journal日志恢复增量的变化。
- 由于Journal日志持久化的间隔非常短，数据能得到更高的保障，
  - 如果按照当前版本的默认配置，则其在断电情况下最多会丢失100ms的写入数据。

![](https://note.youdao.com/yws/public/resource/c1fdfae65ce29ffac56d7315bbfc7079/xmlnote/8320B8C17D22456E89EB1F9099FCD1E4/43547)

##### WriedTiger写入数据的流程

1. 应用向MongoDB写入数据（插入、修改或删除）
2. 数据库从内部缓存中获取当前记录所在的页块，如果不存在则会从磁盘中加载（Buffer I/O）
3. WiredTiger开始执行写事务，修改的数据写入页块的一个更新记录表，此时原来的记录仍然保持不变
4. 如果开启了Journal日志，则在写数据的同时会写入一条Journal日志（Redo Log）
   1. 该日志在最长不超过100ms之后写入磁盘
5. 数据库每隔60s执行一次CheckPoint操作，此时内存中的修改会真正刷入磁盘

