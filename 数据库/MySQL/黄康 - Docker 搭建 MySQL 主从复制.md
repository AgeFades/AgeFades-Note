# 黄康 - Docker 搭建 MySQL 主从复制

```shell
# 创建 master 挂载目录
mkdir -p /docker/mysql-master/conf
mkdir -p /docker/mysql-master/data

# 修改字符编码
vim /docker/mysql-master/conf/my.cnf

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

# 启动 master
docker run -p 13307:3306 \
--name mysql-master \
-e MYSQL_ROOT_PASSWORD=agefades \
--privileged=true \
-v /docker/mysql-master/data:/var/lib/mysql \
-v /docker/mysql-master/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

```shell
# 创建 Slave 挂载目录
mkdir -p /docker/mysql-slave/conf
mkdir -p /docker/mysql-slave/data

# 修改字符编码
vim /docker/mysql-slave/conf/my.cnf

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

# 启动 Slave
docker run -p 13308:3306 \
--name mysql-slave \
-e MYSQL_ROOT_PASSWORD=agefades \
--privileged=true \
-v /docker/mysql-slave/data:/var/lib/mysql \
-v /docker/mysql-slave/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

```shell
# 进行主从操作

# 开启 master 的 binlog 日志
vim /docker/mysql-master/conf/my.cnf

# 添加的配置
[mysqld]
## 同一局域网内注意要唯一
server-id=100  
## 开启二进制日志功能，可以随便取（关键）
log-bin=mysql-bin

# 保存重启 master
docker restart mysql-master

# 进入容器内部
docker exec -it mysql-master bash
mysql -u root -p
agefades

# 创建用户
use mysql
CREATE USER 'slave'@'%' IDENTIFIED BY 'agefades';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
```

``` shell
# 进行主从操作

# 修改 slave 配置
vim /docker/mysql-slave/conf/my.cnf

[mysqld]
## 设置server_id,注意要唯一
server-id=101  
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=mysql-slave-bin   
## relay_log配置中继日志
relay_log=edu-mysql-relay-bin  

# 重启容器
docker restart mysql-slave
```

```shell
# 先去主机 MySQL 中查询 binlog 日志的信息
show master status

# 该信息要在 slave 中用到
change master to master_host='39.108.158.31', master_user='slave', master_password='123456', master_port=3301, master_log_file='mysql-bin.000001', master_log_pos= 154, master_connect_retry=30;

# 创建 slave 配置详情
master_host # Master的地址，指的是容器的独立ip,可以通过docker inspect --
master_port # Master的端口号，指的是容器的端口号
master_user # 用于数据同步的用户
master_password # 用于同步的用户的密码
master_log_file # 指定 Slave 从哪个日志文件开始复制数据，即上文中提到的 File 字段的值
master_log_pos # 从哪个 Position 开始读，即上文中提到的 Position 字段的值
master_connect_retry # 如果连接失败，重试的时间间隔，单位是秒，默认是60秒

# 查看状态
show slave status 

# 启动 slave
start slave

# 再次查看状态
show slave status

# 出现以下则成功
Slave_IO_Running=true
Slave_SQL_Running=true

# 主创建测试表，插入数据，检查从表是否有
```

