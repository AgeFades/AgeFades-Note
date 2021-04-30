[TOC]

# 杨过 - 一条SQL的执行过程

## MySQL 的内部组件结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598522414632.png)

```shell
# 大体上来说，MySQL 可以分为 Server层 和 存储引擎层 两部分。
```

### Server 层

```shell
# 主要包括:
	# 连接器
	
	# 查询缓存
	
	# 分析器
	
	# 优化器
	
	# 执行器
	
# Server 层涵盖了 MySQL 的大多数核心服务功能，
	# 以及所有的内置函数（如 日期、时间、数学、加密函数...）,
	
	# 所有 跨存储引擎的功能 都在 Server 层实现，
		# 比如: 存储过程、触发器、视图...
```

#### 连接器

##### 简介

```shell
# MySQL 有非常多种类的客户端，
	# 比如: navicat、mysql front、jdbc、SQL youg ...
	
# 所有的客户端，要与 MySQL 进行通信，必须先跟 Server 端建立通信连接，
	# 而建立连接的工作，就是由 连接器 完成的。
	
# 连接器 负责跟 客户端 建立连接、获取权限、维持和管理连接。
```

```sql
-- 连接命令一般格式
mysql -h[host] -u[user] -p[pwd] -P[port]

-- 示例
mysql -hlocalhost -uroot -proot -P3306
```

##### 认证

```shell
# 在完成经典的 TCP 握手连接后，连接器就会开始 认证 客户端命令的身份
	# 如果用户名或密码不对，
		# 客户端收到 "Access denied for user" 错误，程序结束，连接关闭。
		
	# 如果认证通过，连接器 到 权限表 里查询该用户拥有的 权限，
		# 之后，该 连接会话 里的 权限判断逻辑，都依赖于 此时读到的权限。
		
		# 这就意味着，一旦 连接建立成功，即时管理员对该用户权限进行了修改，
			# 也不会影响现有连接会话的权限，
			
			# 只会在之后 新建的连接会话 使用新的权限设置

# 用户的权限表是 mysql 中的 user 表
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598524314779.png)

##### 常见权限语句

```sql
-- 创建新用户
create user 'username'@'host' identified by 'password';

-- 赋予权限，% 表示所有(host)
grant all privileges on *.* 'username'@'%';

-- 刷新数据库
flush privileges

-- 设置用户名密码
update user set password=password("123456") where user = "root";

-- 查看当前用户的权限
show grants for root@"%";
```

##### 连接状态

```shell
# 连接完成，没有其余操作的话，该连接就处于空闲状态。

# 查看连接会话列表
show processlist
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598844889888.png)

```shell
# 客户端如果一直长时间处于空闲状态，连接器就会自动将该连接断开。

# 空闲会话超时自动关闭时间参数、默认值 8 小时（28800秒）
wait_timeout
```

```sql
-- 查看 空闲会话超时自动关闭时间
show global variables like "wait_timeout";

-- 设置全局 空闲会话超时自动关闭时间（24小时）
set global wait_timeout = 86400
```

```shell
# 连接被断开后，客户端再次发送请求，连接器返回错误响应，需要重新连接
Lost connection to MySQL server during query
```

##### 关于MySQL OOM

```shell
# MySQL中，如果客户端持续有请求，则一直使用同一个连接，即 长连接。
	# 短连接 是每次执行完很少的 SQL 就断开连接，下次查询再重新建立一个。
	
# 开发中，大多数是长连接，放在 数据库连接池 中进行管理，
	# 而由于MySQL在 执行SQL 中，临时使用到的内存都是在连接对象里面的，
	
	# 这些内存只会在连接断开的时候才释放，
	
	# 所以，长时间下来，长连接可能导致 会话内存占用过大，被系统强行 Kill，
	
	# 从 现象 上来看，就是 MySQL 异常重启。
```

#### 查询缓存

```shell
# 鸡肋，MySQL 8.0 已经移除。
```

#### 分析器

```shell
# 分析器负责对 SQL 语句进行解析。
```

```shell
# SQL分析的 6 个主要步骤:
1. 词法分析

2. 语法分析

3. 语义分析

4. 构造执行树

5. 生成执行计划

6. 计划的分析
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598853009936.png)

```shell
# 分析器首先做的就是对 SQL 语句做 词法分析，
	# SQL 由 多个字符串和空格 组成，
	
	# 分析器需要识别出里面的 字符串 分别是什么，代表什么,
		# 比如: 根据 select 分析出是一个查询语句，from t1 识别成 "表t1"
		
# 词法分析 主要由 MySQLLex 完成。
```

```shell
# 完成了 词法分析 后，就是 语法分析。

# 根据 语法规则，判断该 SQL 语句是否符合 MySQL 语法。

# 如果语法不对，服务端响应错误提示
You have an error in you SQL syntax

# 语法分析主要由 Bison 完成、然后生成 语法树
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598852826146.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598853192510.png)

#### 优化器

```shell
# 经过 分析器 之后， MySQL 知道了 SQL 将要做什么，
	# 在开始执行之前，还会经过 优化器 的处理。
	
# 优化器是在表中有 多个索引 时，决定使用 某个索引，
	# 举例: where a = 1 and b = 1，优化器会根据成本计算，决定使用哪个索引。
	
# 或者在 多表关联(join) 时，决定 各个表的连接顺序。
	# 举例: t1、t2 关联查询，优化器会选择其中 小表驱动大表（可能改变原SQL顺序）
```

#### 执行器

```shell
# sql 执行之前，会先判断 该用户有没有对该表执行该操作 权限，
	# 如果有，打开表，根据 表的引擎 定义，调用 该引擎提供的接口 执行。
	
# 比如, test 表，name 字段没有索引，执行流程如下:
1. 调用 Innodb 引擎接口取该表的第一行，判断 name 是否等于查询条件，不是则跳过，是则将这行记录存放在结果集中。

2. 调用 Innodb 引擎接口取 下一行，重复的判断逻辑，全表扫描。

3. 执行器将 结果集 返回给客户端。
```

### 存储引擎层

```shell
# 主要负责 数据的存储和提取。

# 采取 插拔式 的架构模式。

# 支持 InnoDB、MyISAM、Memory...
	# MySQL 5.5.5 版本开始，默认存储引擎为 InnoDB.
```

## bin-log

[参考文章](https://www.jianshu.com/p/8e7e288c41b1)

```shell
# binlog 是 Server 层实现的二进制文件，记录客户端的 crud 操作。

1. Binlog 在 Server 层实现（引擎共用）

2. Binlog 为逻辑日志，记录的是一条 SQL 的原始逻辑。

3. Binlog 不限大小，追加写入，不会覆盖以前的日志。
```

### 配置

```properties
# 配置开启 binlog
log-bin=${binlog日志文件 存放目录}
# 5.7+ 需要配置本项，保证唯一性（自定义）
server-id=Swordsman
# binlog 格式: statement、row、mixed
binlog-format=ROW
# 每次执行就与系统同步，会有一定影响性能
# 为 0 时表示，事务提交时，MySQL 不做 IO 操作，由系统决定
sync-binlog=1
```

### 命令

```sql
-- 查看 bin-log 是否开启
show variables like '%log_bin%';

-- 刷新 bin-log 日志，生成新文件存储
flush logs;

-- 查看最后一个 bin-log 日志的相关信息
show master status;

-- 清空所有的 bin-log 日志
reset master;

-- 查看 bin-log 内容
mysqlbinlog --no-defaults binlog.000001(具体的 binlog 文件名)
```

### 数据恢复

```shell
# bin-log 内容不具备可读性，重点看 begin-commit(一个完成的事务逻辑),
	# 再根据位置 position 判断恢复即可。
```

```sql
-- binlog.000001 替换成具体的 binlog 文件
-- db 替换成具体的数据库

-- 恢复全部数据
mysqlbinlog --no-defaults binlog.000001 | mysql -uroot -p db

-- 恢复指定位置数据
mysqlbinlog --no-defaults --start-position="408" --stop-position="731" binlog.00001 | mysql -uroot -p db

-- 恢复指定时间段数据
mysqlbinlog --no-defaults --start-date="2020-08-31 00:00:00" --stop-date="2020-09-01 00:00:00" binlog.00001 | mysql -uroot -p db
```

