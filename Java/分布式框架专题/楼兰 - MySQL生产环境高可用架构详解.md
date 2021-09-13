[TOC]

# 楼兰 - MySQL生产环境高可用架构详解

## 实验目的与环境

### 实验目的

- 企业项目开发中，生产环境数据库必须实现高可用（预防单点故障）
- 此篇用于了解MySQL生产环境下架构

### 实验环境

- 两台服务器，分别部署一个MySQL8实例
  - 这里测试一主多从、互主模式等，用了三台

| 节点   | ip            | port |
| ------ | ------------- | ---- |
| 主节点 | 192.168.1.12  | 3306 |
| 从节点 | 192.168.1.28  | 3306 |
| 从节点 | 192.168.1.115 | 3306 |

## 基础环境

- 为了便于使用，两个MySQL实例均需打开远程登录权限

- 开启方式：在客户端执行SQL语句

```sql
#开启远程登录
use mysql;
update user set host='%' where user='root';
flush privileges; 
```

## 搭建主从集群

### 理论基础

- 通过搭建MySQL主从集群，缓解MySQL读写压力

#### 数据安全

- 给主服务增加一个数据备份
  - 基于该目的，可以搭主从、也可以搭基于主从的互主架构

#### 读写分离

- 一般来说，读请求 > 写请求，于是可以 **主写从读**
  - MySQL主从架构只是实现读写分离的一个基础，还需要一些中间件支持
  - 如：ShardingSphere

#### 故障转移

- 即 **高可用**，主节点挂了，从节点切成主节点 顶上读写请求
  - **主从数据同步** 也是实现高可用的一个基础
  - 要想实现 **主从切换**，仍需一些中间件支持
  - 如：MMM、MHA、MGR

### 同步原理

- MySQL主从一般都是通过 **binlog** 日志文件进行
  - 主库开启 **binlog** 用于记录每一步数据库操作
  - -> 从库开一个 **IO线程** 跟主服务建立一个 **TCP** 链接，请求主库将 **binlog** 传输过来
  - -> 主库开一个 **IO dump线程**，负责通过该 **TCP** 链接，将 **binlog** 传输给从库的 **IO线程**
  - -> 从库将 **IO线程** 读到的 **binlog** 日志数据写入自己的 **relay日志**
  - -> 从库另起一个 **SQL线程** 读取 **relay日志** 内容，进行 **操作重演**，完成 **数据同步**

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/6B6590A632734A45A96B97FC8DAA8CC7?ynotemdtimestamp=1631241134004)

#### binlog其余场景

- binlog 不光可以用于 主从同步，还可以用于 缓存数据同步 等场景
  - 如：**Canal**
  - 模拟一个从节点
  - -> 向主库发起 **binlog同步请求**
  - -> 将数据同步到 Redis、Kafka 等其他中间件
  - -> 实现数据实时流转

#### 搭建主从前提

- 双方MySQL **必须版本一致**
  - 至少要 主库版本 < 从库版本
- 两节点间的时间需要同步

### 搭建步骤

#### 1. 建立docker网段

- 方便后续可能接入的 Sharding Sphere 组件互通

```shell
# 两台服务器都执行
docker network create --subnet=173.18.0.0/24 agefades
```

#### 2. 创建MySQL8容器

##### 1. 创建挂载目录

```shell
# 查看当前目录(自己找个目录，示例如后)，/root/agefades/mysql
# 两台服务器都执行
pwd

mkdir data datadir config
```

##### 2. 编辑主库配置文件

```shell
vim config/my.cnf
```

```apl
[mysqld]
server-id=47
# 开启binlog
log_bin=master-bin
log_bin-index=master-bin.index
skip-name-resolve
# 设置连接端口
port=3306
# 设置mysql的安装目录
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql/mysql-files
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。
max_connect_errors=10
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 默认使用“mysql_native_password”插件认证
#mysql_native_password
default_authentication_plugin=mysql_native_password
```

- 主要修改以下几个属性：
  - server-id：服务节点的唯一标识，需要给集群中的每个服务分配一个单独的id
  - log_bin：打开 binlog 日志记录，并指定文件名
  - log_bin-index：binlog日志文件

##### 3. 编辑从库配置文件

```shell
vim config/my.cnf
```

```apl
[mysqld]
# 主库和从库需要不一致
server-id=48
# 打开MySQL中继日志
relay-log-index=slave-relay-bin.index
relay-log=slave-relay-bin
# 打开从服务二进制日志
log-bin=mysql-bin
# 使得更新的数据写进二进制日志中
log-slave-updates=1
# 设置3306端口
port=3306
# 设置mysql的安装目录
basedir=/usr/local/mysql
# 设置mysql数据库的数据的存放目录
datadir=/usr/local/mysql/mysql-files
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。
max_connect_errors=10
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 默认使用“mysql_native_password”插件认证
# mysql_native_password
default_authentication_plugin=mysql_native_password
```

- 主要关注以下属性：
  - server-id：服务节点的唯一标识
  - relay-log：打开从服务的 relay-log 日志
  - log-bin：打开从服务的 bin-log 日志记录

##### 4. docker-compose启动

- **两台服务器都执行**

- 编写 docker-compose 文件

```shell
vim docker-compose-mysql.yaml
```

```yaml
version: '3.4'
services:
  mysql:
    container_name: mysql
    hostname: mysql
    image: mysql:8.0.20
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_ROOT_HOST: "%"
    volumes:
      - ./data:/var/lib/mysql
      - ./datadir:/usr/local/mysql
      - ./config:/etc/mysql/conf.d
    command:
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_unicode_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M
      --default-time-zone='+8:00'
    networks:
      agefades:
        ipv4_address: 173.18.0.10
networks:
  agefades:
    external: true
```

- 编写简易部署文件

```shell
vim deploy.sh
```

```shell
# deploy.sh 内容
docker-compose -f docker-compose-mysql.yaml down
docker-compose -f docker-compose-mysql.yaml up -d
```

- 执行部署

```shell
sh deploy.sh
```

#### 3. 打开远程登录权限

- 两台服务器都执行

```shell
# 进入容器
docker exec -it mysql bash

# 进入MySQL客户端
mysql -uroot -proot
```

```sql
# 开启远程登录
use mysql;
update user set host='%' where user='root';
flush privileges;
```

- 这里创建的容器好像默认就开启了，可以自行查看

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631246069912.png)

#### 4. 给主库分配权限

- 需要给主库用户分配一个 replication slave 的权限
  - 实际生产环境中，通常不会直接使用 root 用户，
  - 而是创建一个拥有全部权限的单独用户来负责主从同步

```sql
# 分配权限
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%';
# 刷新权限
flush privileges;
# 查看主节点同步状态
show master status;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631246257737.png)

- 该指令结果：
  - FIle：当前 binlog 日志文件
  - Position：当前 binlog 日志文件索引
  - Binlog_Do_DB：需要记录 binlog 文件的库
  - Binlog_Ignore_DB：不需要记录 binlog 文件的库
- Binlog_Do_DB 和 Binlog_Ignore_DB 没配置时，
  - 表示针对全库记录日志
- 开启 binlog 后，数据库所有操作都会被记录到 datadir 中
  - 以 **一组轮询文件** 的方式循环记录
  - 后面配置从库时，就需要通过 **File 和 Position** 获取从哪个地方开始记录 binlog

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631247809541.png)

#### 5. 给从库配置主节点同步

- 注意根据自己操作情况修改以下属性设置
- 注意：
  - MASTER_LOG_FILE、MASTER_LOG_POS
  - 这两个属性必须与主库查到的保持一致

```sql
# 设置同步主节点
CHANGE MASTER TO
MASTER_HOST='192.168.1.12',
MASTER_PORT=3306,
MASTER_USER='root',
MASTER_PASSWORD='root',
MASTER_LOG_FILE='master-bin.000003',
MASTER_LOG_POS=943,
GET_MASTER_PUBLIC_KEY=1;

# 开启slave
start slave;

# 查看主从同步状态
show slave status \G;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631248374832.png)

- 后续如果要检查主从架构是否成功，也可以通过检查主库和从库的
  - File、Position 这两个属性是否一致来确定
- 同步哪些库、哪些表，忽略同步哪些库、哪些表的配置
  - Replicate_Do_DB
  - Replicate_Ignore_DB
  - Replicate_Do_Table
  - Replicate_Ignore_Table
  - Replicate_Wild_Do_Table
  - Replicate_Wild_Ignore_Table

#### 6. 主从同步测试

##### 1. 查看数据库情况

- 两个库都执行

```sql
show dabatases;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631248756891.png)

##### 2. 主库创建库

```sql
create database syncdemo;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631249067488.png)

##### 3. 主库创建表插数据

```sql
use syncdemo;

create table demoTable(id int not null);

insert into demoTable value(1);
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631249215448.png)

#### 7. 主从搭建完成

- 从以上实验看出，主库操作都已成功同步到从库

#### 8. 异常情况

##### 现象

- 在从服务上，查看 slave 状态，发现 Slave_SQL_Running=no
  - 即表示主从同步失败

##### 原因

1. 在从库进行了写操作，与同步过来的SQL操作冲突了
2. 从库重启后，有事务回滚了

##### 解决思路

- 如果是因为从库事务回滚的原因，可以按照以下方式重启主从同步

```sql
stop slave;
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave;
```

- 另一种解决方案就是重新记录主节点的binlog文件信息
  - 这种方式如果修改后和之前的 File、Position 对不上，就会丢失部分数据
  - 所以这种方式不太常用

```sql
stop slave;
change master to .....
start slave;
```

### 集群搭建扩展

#### 全库同步与部分同步

##### 主库配置

- 通过以下属性指定哪些库或哪些表记录 binlog

```shell
vim config/my.cnf
```

```apl
# 需要同步的二进制数据库名
binlog-do-db=masterdemo
# 只保留7天的二进制日志，以防磁盘被日志占满(可选)
expire-logs-days  = 7
# 不备份的数据库
binlog-ignore-db=information_schema
binlog-ignore-db=performation_schema
binlog-ignore-db=sys
```

##### 从库配置

```shell
vim config/my.cnf
```

```apl
# 如果salve库名称与master库名相同，使用本配置
replicate-do-db = masterdemo 
# 如果master库名[mastdemo]与salve库名[mastdemo01]不同，使用以下配置[需要做映射]
replicate-rewrite-db = masterdemo -> masterdemo01
# 如果不是要全部同步[默认全部同步]，则指定需要同步的表
replicate-wild-do-table=masterdemo01.t_dict
replicate-wild-do-table=masterdemo01.t_num
```

##### 查看配置

```sql
# 查看 Binlog_Do_DB 和 Binlog_Ignore_DB 两个参数的作用
show master status
```

#### 读写分离配置

##### 问题

- 目前MySQL主从复制是单向的，只能 主 -> 从
  - 这种架构下，为保证数据一致，通常需要做 **主写从读、读写分离**
- 注意，MySQL主从本身无法提供读写分离的服务，需要由业务自己实现，比如后面要学的 ShardingSphere

##### 解决思路

- 如果需要限制用户写数据，
  - 可以在从服务中将 read_only 参数的值设为1
  - set global read_only = 1;
  - 这样就可以

- 注意：
  1. read_only=1 不会影响主从复制的功能
  2. 只限定了普通用户写数据，不会限定 super 权限的用户
     1. 如果需要限定 super 权限用户写数据，可以设置 super_read_only=0
     2. 禁止 super 权限用户写操作，flush tables with read lock;
     3. 这样设置也会阻止主从复制

#### 其他集群方式

- 这里搭建的 一主一从 ，具备数据同步的基础功能，
  - 生产环境中，通常以此为基础，根据业务及负载情况，搭建更大更复杂的集群
- 例如：
  - 一主多从：进一步提高整个集群的读能力
  - 一主多级从：减轻主节点进行数据同步的压力
  - 多主集群：提高整个集群高可用
  - 互主集群 或 环形主从集群：实现MySQL多活部署
- 搭建互主集群：
  - 按照上面方式，再从 主库 打开一个 Slave 线程
  - -> 指向从库的 binlog File、Position

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/F1DEBE74430D41089720C922B164D61D?ynotemdtimestamp=1631241134004)

- 其他方式顾名思义，应该很简单就能发散思维完成

### GTID同步集群

#### 简介

- 上面搭建方式，是基于 binlog 日志记录点的方式，也是最传统的 MySQL 集群搭建方式
- 在 show master status; 命令中，有一个 Executed_Grid_Set 列
  - 暂时没有用上，实际上是另一种搭建 主从复制 的方式
  - 即 GTID 搭建方式，是 MySQL5.6 版本引入的

#### 原理

- GTID 本质也是基于 binlog 来实现主从同步，
  - 但是它会基于一个 **全局的事务ID** 来标识同步进度，
  - **GTID 即 全局事务ID**，全局唯一 且 趋势递增，
  - 可以保证 **为每一个在主节点上提交的事务** 在 **复制集群** 中生成一个唯一的ID
- 在基于 GTID 的复制中，首先 
  - 从库会告诉主库 **执行完的事务GTID值**
  - -> 主库把 **所有从库没执行的事务**，发送到从库上执行
  - -> 并使用 GTID 的复制保证 **同一个事务只在指定的从库上执行一次**，这样可以避免由于偏移量的问题造成数据不一致

#### 搭建方式

- 和上面的 主从架构搭建 整体差不多，只需要在 my.cnf 修改一些配置

##### 主库配置

```apl
gtid_mode=on
enforce_gtid_consistency=on
log_bin=on
server_id=单独设置一个
binlog_format=row
```

##### 从库配置

```apl
gtid_mode=on
enforce_gtid_consistency=on
log_slave_updates=1
server_id=单独设置一个
```

- 两库都重启后，即可开启 GTID 同步复制方式

## 集群扩容

- 目前已成功搭建 一主一从，扩展到 一主多从，只需要增加一个 binlog 复制就行

### 追加扩容

- 如果集群已经运行了一段时间，此时要扩展新的从节点就有一个问题
  - 之前的数据没办法从 binlog 来恢复了，
  - 此时就需要增加一个 **数据复制** 的操作

#### 数据备份

- MySQL的数据备份恢复操作：
  - 使用 mysql bin目录下的 mysqldump 工具

```shell
# 主库,在容器外执行
mysqldump -u root -p root --all-databases > backup.sql
```

##### 问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631255998383.png)

##### 原因

- 开启了 GTID同步 导致

##### 解决

```shell
mysqldump -u root -p root --all-databases --triggers --routines --events > backup.sql
```

#### 数据导入

- 将主库到处的 backup.sql 弄到新的从库服务器中
  - 这里我用的是 Mac 的 scp 命令完成
- 执行下面指令进行导入

```shell
mysql -u root -p < backup.sql
# 输入密码
```

- 同步成功截图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1631257506069.png)

## 半同步复制

### 理解半同步复制

- 目前为止，搭建的集群有 **可能丢数据的隐患**

#### 异步复制

##### 流程机制

- MySQL主从集群默认采用的是一种 **异步复制** 的机制
  - 主库在执行用户提交的事务后，写入 binlog 日志
  - -> 给客户端返回成功响应
  - -> binlog 由一个 dump 线程异步发送给 slave 从库

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/0045756537434AECA30EAA891CAE8D42?ynotemdtimestamp=1631241134004)

##### 问题

- 由于发送 binlog 的过程是异步的，主库给客户端成功响应时，是不知道 binlog 是否同步成功的
  - 此时如果 **主库宕机**，从库还没有备份到新执行的 binlog
  - 就可能会丢失数据

#### 半同步复制

##### 优化机制

- 针对上面问题，可以用MySQL的 **半同步复制机制** 来保证数据安全
- **半同步复制机制**：
  - 一种介于 **异步复制** 和 **全同步复制** 之间的机制
  - 主库执行完客户端提交事务后，并不会立即返回客户端成功响应，
    - -> 而是等待 **至少一个从库** 接收并写到 **relay log** 中
    - -> 才会返回给客户端
  - MySQL在等待确认时，默认会等10秒，如果超过10秒没有收到ack，就会降级成为异步复制

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/5A202AC6CC344F99BDA7A18868A1F9BF?ynotemdtimestamp=1631241134004)

##### 问题

1. 相比 **异步复制**，能够有效提高数据安全性，但这种安全性不是绝对的
   - 它只保证 事务提交后的binlog 至少传输到了一个从库
   - 并且不保证从库应用这个事务的binlog是成功的
2. 半同步复制机制 也会造成 **一定程度的延迟**
   - 这个延迟时间最少是一次 TCP/IP 请求往返的时间
   - 整个服务性能就会下降
   - 而当从库出现问题时，主库给客户端响应时间就会更长

### 搭建半同步复制集群

- 需要特定的扩展模块实现，MySQL5.5之后，默认自带该模块
  - 在 mysql 安装目录下的 lib/plugin 目录
  - 主库模块：semisync_master
  - 从库模块：semisync_slave

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/AE4DA10109D14F1B94E7F71855AF4425?ynotemdtimestamp=1631241134004)

- docker 容器里没找到该插件，搜了一圈（包括官网）没找到怎么从远端下载这些插件模块
  - 可以找个本地安装的 copy 一份该插件模块过去（懒得搞了）
- 下面操作仅按教程记录

#### 主库操作

```apl
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
Query OK, 0 rows affected (0.01 sec)

mysql> show global variables like 'rpl_semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
+-------------------------------------------+------------+
6 rows in set, 1 warning (0.02 sec)

mysql> set global rpl_semi_sync_master_enabled=ON;
Query OK, 0 rows affected (0.00 sec)
```

- 第一行指令：
  - 通过扩展库来安装半同步复制模块，需要指定扩展库的文件名
- 第二行指令：
  - 查看系统全局参数
    - rpl_semi_sync_master_timeout 即等待应答的最长等待时间，默认10秒
    - 可自行调整参数
  - rpl_semi_sync_master_wait_point：表示一种半同步复制的方式
    - 默认 AFTER_SYNC
      - 主库写日志到 binlog
      - -> 复制给从库，等待从库响应
      - -> 从库返回成功后，主库再提交事务，给客户端成功响应
    - AFTER_COMMIT
      - 主库写日志到 binlog
      - -> 等待 binlog 复制到从库，主库就提交自己的本地事务
      - -> 再等待从库返回给自己一个成功响应，主库再给客户端响应
- 第三行指令：
  - 打开半同步复制的开关

#### 从库操作

```apl
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
Query OK, 0 rows affected (0.01 sec)

mysql> show global variables like 'rpl_semi%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| rpl_semi_sync_slave_enabled     | OFF   |
| rpl_semi_sync_slave_trace_level | 32    |
+---------------------------------+-------+
2 rows in set, 1 warning (0.01 sec)

mysql> set global rpl_semi_sync_slave_enabled = on;
Query OK, 0 rows affected (0.00 sec)

mysql> show global variables like 'rpl_semi%';
+---------------------------------+-------+
| Variable_name                   | Value |
+---------------------------------+-------+
| rpl_semi_sync_slave_enabled     | ON    |
| rpl_semi_sync_slave_trace_level | 32    |
+---------------------------------+-------+
2 rows in set, 1 warning (0.00 sec)

mysql> stop slave;
Query OK, 0 rows affected (0.01 sec)

mysql> start slave;
Query OK, 0 rows affected (0.01 sec)
```

## 主从架构的数据延迟问题

### 现象

- 刚插入的数据马上去查，查不到

- **主写从读，读写分离** 下更容易体现这个问题

### 原因

- 面向业务的主库数据，都是 **多线程并发写入**
  - 而从库是 **单个线程慢慢拉取binlog**
  - 中间就有个效率差

### 解决

- 关键就是让 **从库也用多线程并行复制 binlog 数据**

- MySQL5.7版本后，支持并行复制
  - 在从库上设置：
    - slave_parallel_workers 为一个大于0的数
    - slave_parallel_type 设置为 LOGICAL_CLOCK

## MySQL的高可用方案

### 简介

- 此前的MySQL服务集群，都是基于 MySQL 自身功能搭建的集群，不具备高可用性
  - 即：主库挂了，从库没办法自动切换成主库
- 要实现 MySQL 的高可用，需要借助一些第三方工具来实现
- 常见的MySQL集群方案有三种：
  - MMM
  - MHA
  - MGR
- 这三种高可用框架都有一些共同点：
  - 对主从复制集群中的 Master 节点进行监控
  - 通过 **VIP** 自动的对 Master 进行迁移
  - 重新配置集群中的其他 slave 对新的 Master 进行同步

### MMM

#### 简介

- MMM(Master-Master replication managerfor MySQL)
  - MySQL主主复制管理器
- 一套由 Perl 语言实现的脚本程序
  - 可以对MySQL集群进行监控和故障转移
  - **需要两个Master**，同一时间只有一个Master对外提供服务
  - 可以说是主备模式
- 通过一个 **VIP(虚拟IP)** 的机制来保证集群的高可用
  - 整个集群中，主节点上会通过一个 **VIP地址** 来提供数据读写服务
  - 出现故障时，VIP就会从原来的主节点转移到其他节点，从而继续提供服务

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/550A36B2E84E4F559564A249586AEDFD?ynotemdtimestamp=1631241134004)

#### 优点

- 提供读写VIP配置，使读写请求都可以达到高可用
- 工具包相对比较完善，不需要额外的开发脚本
- 完成故障转移之后，可以对MySQL集群进行高可用监控

#### 缺点

- 故障处理简单粗暴，容易丢失事务，建议采用半同步复制方式，减少失败概率
- 目前 MMM 社区已经缺少维护，不支持基于 GTID 的复制

#### 适用场景

- 读写都需要高可用的
- 基于日志点的复制方式

### MHA

#### 简介

- Master High Availability Manager and Tools for MySQL
  - 日本人基于 Perl 脚本写的一个工具
- 专门用于监控主库的状态
  - 发现主库故障时，会提升其中 **拥有新数据的从库** 成为新的主库
  - 在此期间，MHA 会通过其他从节点获取额外的信息来避免数据一致性的问题
- MHA 还提供了 **主库的在线切换功能**，即按需切换 master-slave
  - 能在30秒内实现故障转移，并在转移过程中，最大程度保证数据一致性
  - 淘宝内部有个类似的产品 TMHA
- MHA 需要单独部署，分为 Manager节点 和 Node节点
  - Manager节点一般是一台单独部署的机器
  - Node节点一般是部署在每台 MySQL 机器上
    - Node节点通过解析各个 MySQL  的日志来进行一些操作
- Manager 通过探测集群里的 Node节点，去判断该机器上的MySQL是否正常
  - 如果发现某个 Master 故障，直接把它的一个 Slave 提升为 Master
  - 然后将其他 Slave 都挂向新的 Master

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/47ACB9F2E9304A82A1B949F3B385B5D8?ynotemdtimestamp=1631241134004)

#### 优点

- MHA 除了支持日志点的复制，还支持 GTID 的方式

#### 缺点：

- 同MMM相比，MHA会尝试从旧的Master中恢复旧的二进制日志，只是未必每次都能成功。如果希望更少的数据丢失场景，建议使用MHA架构。
- MHA需要自行开发VIP转移脚本。
- MHA只监控Master的状态，未监控Slave的状态

### MGR

下面都是直接从教案 Copy 过来的

我感觉就是 ZAB、Raft 等分布式一致性协议的思想变种实现

#### 简介

- MySQL Group Replication，MySQL官方在5.7.17版本正式推出的一种组复制机制
  - 主要是解决传统异步复制和半同步复制的数据一致性问题
- 由若干个节点共同组成一个复制组
  - 一个事务提交后，必须经过超过半数节点的决议并通过后，才可以提交
  - 引入组复制，主要是为了解决传统异步复制和半同步复制可能产生数据不一致的问题
- MGR依靠分布式一致性协议(Paxos协议的一个变体)，实现了分布式下数据的最终一致性，提供了真正的数据高可用方案(方案落地后是否可靠还有待商榷)。

- 支持多主模式，但官方推荐单主模式：
  - 多主模式下，客户端可以随机向MySQL节点写入数据
  - 单主模式下，MGR集群会选出primary节点负责写请求，primary节点与其它节点都可以进行读请求处理.

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/6C7F0DBDC2EE489D9DC2717D1DEC9C25?ynotemdtimestamp=1631241134004)

#### 优点

- 基本无延迟，先插后查时，延迟比异步的小很多
- 支持多写模式，但是目前还不是很成熟
- 数据的强一致性，可以保证数据事务不丢失

#### 缺点

- 仅支持innodb，且每个表必须提供主键。
- 只能用在GTID模式下，且日志格式为row格式。

#### 适用场景

- 对主从延迟比较敏感
- 希望对对写服务提供高可用，又不想安装第三方软件
- 数据强一致的场景

## 分库分表

### 作用

- 解决由于 **数据量过大** 而导致数据库性能降低的问题
  - 本人碰到过的需求场景：一套租户数据一个库（物理隔离）
- 例如：
  - 微服务架构中，每个服务一个独立数据库（分库）
  - 一些业务日志表，按月拆成不同的表（分表）

### 方式

- 分库、分表，统称为数据分片
- 可以分为 **垂直分片** 和 **水平分片**

#### 垂直分片

- 核心理念：**专库专用**

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/A4F4BCF99FE14CFAA2369831CEBF3EE6?ynotemdtimestamp=1631241134004)

- 垂直分片通常需要对架构和设计进行调整，通常是来不及应对业务需求快速变化的
  - 可以缓解数据量和访问量带来的问题
  - 但无法真正解决单点数据库的性能瓶颈，单表数据量超过阈值，仍需进行水平分片

#### 水平分片

- 通过某些字段、某些规则，将数据分散到多个库或多个表中
  - 每个分片仅包含数据的一部分

![](https://note.youdao.com/yws/public/resource/df15b9a4c400815870b14a58c25dbb8f/6D10185AFA11449A926275338E4E2A31?ynotemdtimestamp=1631241134004)

##### 常用分片策略

- 取余/取模
  - 优点：均匀存放数据
  - 缺点：扩容非常麻烦（得把数据全部搞出来，重新取模分片）
- 按照范围分片
  - 优点：好扩容
  - 缺点：数据分布不均匀
- 按照时间分片
  - 容易将热点数据区分出来
- 按照枚举值分片
  - 例如：按地区分片
- 按照目标字段前缀指定进行分片
  - 自定义业务规则分片

##### 简单说明

- 水平分片，理论上突破了单机数量处理的瓶颈
  - 扩展相对自由，是分库分表的标准解决方案
- 一般在系统设计阶段，就应该根据业务耦合松紧，来确定垂直分库、垂直分表方案
  - 在数据量及访问压力不大时，首先考虑 缓存、读写分离、索引技术等方案
  - 若数据量极大，且持续增长，再考虑水平分库、水平分表方案

### 缺点

- 事务一致性问题
  - 分库分表后，数据分布在不同库甚至不同服务器，不可避免带来分布式事务问题
- 跨节点关联查询问题
  - 表被分散到不同库，无法进行关联查询
  - 此时需要将 关联查询 拆分成多次查询，然后将获得的结果进行拼装
- 跨节点分页、排序函数
  - 跨节点多库进行 limit分页、order by排序等查询时，需要现在不同的分片中将 数据进行排序并返回，然后将不同分片返回的结果集，进行汇总和再次排序
- 主键避重问题
  - 需要单独设计全局主键，以避免跨库主键重复问题
- 公共表处理
  - 像 参数表、字典表 等公共表，每个数据库都要保存一份
  - 并且对公共表的操作，都要分发到所有分库去执行
- 逻辑复杂度
  - 开发人员、运维人员，需要知道路由键及分片规则
  - 才能找到数据最终落到哪个位置

### 什么时候需要分库分表

- 阿里建议MySQL单表数据达到 500w 或 2G，就进行分库分表
- 业务需要做数据的物理隔离时
- 其实能不用就不用，用了问题多
  - 可以使用本身就支持分布式、数据分片的产品，如 Tidb、PgSQL、ES、....

### 常见的分库分表组件

#### Sharding Sphere

[Sharding Sphere](https://shardingsphere.apache.org/document/current/cn/overview/)

- Sharding-JDBC是当当网研发的开源分布式数据库中间件，他是一套开源的分布式数据库中间件解决方案组成的生态圈，它由Sharding-JDBC、Sharding-Proxy和Sharding-Sidecar（计划中）这3款相互独立的产品组成。 他们均提供标准化的数据分片、分布式事务和 数据库治理功能，可适用于如Java同构、异构语言、容器、云原生等各种多样化的应用场景。

#### MyCat

[MyCat](http://www.mycat.org.cn/)

- 基于阿里开源的Cobar产品而研发，Cobar的稳定性、可靠性、优秀的架构和性能以及众多成熟的使用案例使得MYCAT一开始就拥有一个很好的起点，站在巨人的肩膀上，我们能看到更远。业界优秀的开源项目和创新思路被广泛融入到MYCAT的基因中，使得MYCAT在很多方面都领先于目前其他一些同类的开源项目，甚至超越某些商业产品。

#### DBLE

[DBLE](https://opensource.actionsky.com/)

- 该网站包含几个重要产品。其中分布式中间件可以认为是MyCAT的一个增强版，专注于MySQL的集群化管理。另外还有数据传输组件和分布式事务框架组件可供选择。

