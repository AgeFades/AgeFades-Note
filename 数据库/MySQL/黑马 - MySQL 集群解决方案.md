# 黑马 - MySQL 集群解决方案

## 问题

```shell
# 系统架构中，DBServer 如果是单节点服务，面对大并发、海量数据存储等，问题就会凸显出来。
```

## MySQL 数据库的集群方案

### 读写分离架构

```shell
# 一般来说，DB 都是 "读多写少"
# 所以，有思路 ->
	# 一个主库<写操作>
	# 多个从库<读操作>
	
# 自己区分读写操作吗？连接几个库呢？
```

### 中间件

![UTOOLS1568958155635.png](https://i.loli.net/2019/09/20/lursq9jB3zALOoP.png)

```shell
# 中间件挂了怎么办？
```

### PXC 集群架构

```shell
# PXC 提供读写强一致性的功能，可以保证数据在任何一个节点写入的同时，可以同步到其他节点
	# 以为着可以存其他的任何节点进行读取操作，无延迟。
```

![UTOOLS1568958293809.png](https://i.loli.net/2019/09/20/gYyQVXkvKAlRF24.png)

### 混合架构

![UTOOLS1568958362525.png](https://i.loli.net/2019/09/20/shmHeuLc8I6pkvq.png)

## 搭建主从复制架构

### 主从复制原理

![UTOOLS1568958401735.png](https://i.loli.net/2019/09/20/M1bHmp3n8X2NLQ4.png)

```shell
# master 将数据改变记录到二进制日志(binary log) 中
	# slave 将 master 的 binary log events 拷贝到自己的中继日志 relay log
	# slave 重做中继日志中的事件，将改变反映它自己的数据 <数据重演>
	
# 注意
	# master 和 slave 版本一致
	# master 和 slave 数据一致
	# master 开启二进制文件，master 和 slave 的 server_id 都必须唯一
```

### Master 配置文件

```shell
# vim my.conf

# 开启主从复制，Master 的配置
log-bin = mysql-bin
# 指定 Master 的 server_id
server-id=1
# 指定同步数据库，不指定则同步全部数据库
binlog-do-db=my_test

# 执行 SQL 语句查询状态
SHOW MASTER STATUS
```

### Master 创建同步用户

```shell
# 授权用户 root 密码 root
grant replication slave on *.* to 'root'@'127.0.0.1' identified by 'root';

# 刷新配置
flush privileges;
```

### 从库配置文件

```shell
# vim my.conf
# 指定 server_id，不重复即可
server_id=2

# 以下执行 SQL
CHANGE MASTER TO
	master_host='127.0.0.1',
	master_port=3306,
	master_user='root',
	master_password='root',
	master_log_file='mysql-bin.000006',
	master_log_pos=1120;
	
# 启动 Slave 同步
START SLAVE;

# 查看同步状态
SHOW SLAVE STATUS;
```

