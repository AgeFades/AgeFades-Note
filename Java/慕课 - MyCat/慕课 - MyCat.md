# 慕课 - MyCat

## MyCat 入门

### 	什么是 MyCat

##### 		对DBA :

​			MyCat 相当于 MySQL Server 层

​			MySQL 相当于 MyCat 存储层

​	 		MyCat 不存储数据，所有数据存储在 MySQL 中

##### 		对研发 :

​			MyCat 就是 MySQL

​			MyCat 对使用的 SQL 有一些限制

##### 		对架构 :

​			MyCat 是一个数据库中间层

​			MyCat 可以实现对后端数据库的分库分表和读写分离

​			MyCat 对前端应用隐藏了后端数据库的存储逻辑

### ​	MyCat 的主要作用

#### 		作为分布式数据库中间层使用

#### 	![UTOOLS1562062153610.png](https://i.loli.net/2019/07/02/5d1b2d4aa9a2079164.png)

#### 		实现后端数据库的读写分离及负载均衡

#### ![UTOOLS1562062212689.png](https://i.loli.net/2019/07/02/5d1b2d84daa8c70509.png)

#### 		对业务数据库进行垂直切分

#### 	![UTOOLS1562062388472.png](https://i.loli.net/2019/07/02/5d1b2e34c2a3110726.png)

#### 		对业务数据库进行水平切分

#### ![UTOOLS1562062456848.png](https://i.loli.net/2019/07/02/5d1b2e7930c4c54056.png)

#### 		控制数据库连接的数量

#### ![UTOOLS1562062522362.png](https://i.loli.net/2019/07/02/5d1b2ebaaf79819547.png)

### 	MyCat 的基本元素

#### 		逻辑库				

​			对应用来说相当于 MySQL 中的数据库

​			逻辑库可对应后端多个物理数据库

​			逻辑库中并不保存数据

#### 		逻辑表

​			基本同上

##### 			类别

​				分片表与非分片表按是否被分片划分

​				全局表，在所有分片中都存在的表

​				ER关系表，按 ER 关系进行分片的表

## MyCat 安装

### 	安装步骤图解

![UTOOLS1562062975336.png](https://i.loli.net/2019/07/02/5d1b30804621c90262.png)

