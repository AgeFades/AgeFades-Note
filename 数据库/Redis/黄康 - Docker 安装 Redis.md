# 黄康 - Docker 安装 Redis

## 单机版

```shell
# 简易
docker run -d --name redis -p 6379:6379 redis

# 带密码加挂载数据
docker run -d \
--name redis \
-p 6379:6379 \
-v /docker/redis/data:/data \
redis --requirepass 'agefades'
```

## 生产版

```shell
# 首先创建挂载文件
mkdir -p /docker/redis/{data,conf}

# 编辑配置文件，注意我此处设置了密码，请修改为自己的密码

vim /docker/redis/conf/redis.conf

# 退出后授予权限，因为配置文件执行需要权限
chmod 777 /docker/redis/conf/redis.conf
-----------------------------------------------------

# Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
daemonize no

# 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
# pidfile /var/run/redis.pid

# 指定Redis监听端口，默认端口为6379，作者在自己的一篇博文中解释了为什么选用6379作为默认端口，
# 因为6379在手机按键上MERZ对应的号码，而MERZ取自意大利歌女Alessia Merz的名字
port 16279

# 绑定的主机地址
# bind 0.0.0.0

# 当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
timeout 300

# 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，
# 默认为verbose
loglevel verbose

# 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志
# 记录方式为标准输出，则日志将会发送给/dev/null
logfile stdout

# 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
databases 16

# 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
# save <seconds> <changes>
# Redis默认配置文件中提供了三个条件：
save 900 1
save 300 10
save 60 10000
# 分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。

# 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，
# 可以关闭该选项，但会导致数据库文件变的巨大
rdbcompression yes

# 指定本地数据库文件名，默认值为dump.rdb
dbfilename dump.rdb

# 指定本地数据库存放目录
dir /data

# 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
# slaveof <masterip> <masterport>

# 当master服务设置了密码保护时，slav服务连接master的密码
# masterauth agefades

# 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
# requirepass agefades

# 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，
# 如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端
# 返回max number of clients reached错误信息
maxclients 128

# 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，
#当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，
#Value会存放在swap区
#maxmemory <bytes>

# 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。
#因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
appendonly no

# 指定更新日志文件名，默认为appendonly.aof
appendfilename appendonly.aof

# 指定更新日志条件，共有3个可选值： 
# no：表示等操作系统进行数据缓存同步到磁盘（快） 
# always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
# everysec：表示每秒同步一次（折衷，默认值）
appendfsync everysec
```

```shell
# 启动容器
docker run -d \
--name redis \
-p 16279:6379 \
-v /docker/redis/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
```

## 集群版

```shell
# 首先我们先把所有的挂载目录给创建出来
mkdir -p /docker/redis-cluster/redis{1,2,3,4,5,6}/{data,conf}
```

```shell
# 将下述配置文件放入每个 redis conf配置中

# Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
daemonize no

# 当Redis以守护进程方式运行时，Redis默认会把pid写入/var/run/redis.pid文件，可以通过pidfile指定
#pidfile /var/run/redis.pid

# 指定Redis监听端口，默认端口为6379
# 我们将6个redis设置为1637   1-6的端口
port 16371

# 绑定的主机地址
#bind 0.0.0.0

# 当 客户端闲置多长时间后关闭连接，如果指定为0，表示关闭该功能
timeout 300

# 指定日志记录级别，Redis总共支持四个级别：debug、verbose、notice、warning，
#默认为verbose
loglevel verbose

# 日志记录方式，默认为标准输出，如果配置Redis为守护进程方式运行，而这里又配置为日志
#记录方式为标准输出，则日志将会发送给/dev/null
logfile stdout

# 设置数据库的数量，默认数据库为0，可以使用SELECT <dbid>命令在连接上指定数据库id
databases 16

# 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
#save <seconds> <changes>
#Redis默认配置文件中提供了三个条件：
save 900 1
save 300 10
save 60 10000
# 分别表示900秒（15分钟）内有1个更改，300秒（5分钟）内有10个更改以及60秒内有10000个更改。

# 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩，如果为了节省CPU时间，
#可以关闭该选项，但会导致数据库文件变的巨大
rdbcompression yes

# 指定本地数据库文件名，默认值为dump.rdb
dbfilename dump.rdb

# 指定本地数据库存放目录
dir /data

# 设置当本机为slav服务时，设置master服务的IP地址及端口，在Redis启动时，它会自动从master进行数据同步
#slaveof <masterip> <masterport>

# 当master服务设置了密码保护时，slav服务连接master的密码
masterauth agefades

# 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过AUTH <password>命令提供密码，默认关闭
requirepass agefades

# 设置同一时间最大客户端连接数，默认无限制，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，
#如果设置 maxclients 0，表示不作限制。当客户端连接数到达限制时，Redis会关闭新的连接并向客户端
#返回max number of clients reached错误信息
maxclients 128

# 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，
#当此方法处理 后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。Redis新的vm机制，会把Key存放内存，
#Value会存放在swap区
#maxmemory <bytes>

# 指定是否在每次更新操作后进行日志记录，Redis在默认情况下是异步的把数据写入磁盘，如果不开启，可能会在断电时导致一段时间内的数据丢失。
#因为 redis本身同步数据文件是按上面save条件来同步的，所以有的数据会在一段时间内只存在于内存中。默认为no
appendonly no

# 指定更新日志文件名，默认为appendonly.aof
appendfilename appendonly.aof

# 指定更新日志条件，共有3个可选值： 
#no：表示等操作系统进行数据缓存同步到磁盘（快） 
#always：表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全） 
#everysec：表示每秒同步一次（折衷，默认值）
appendfsync everysec

#开启集群
cluster-enabled yes
```

```shell
# 启动集群

docker run -d \
--name redis1 \
-p 16371:6379 \
-p 26371:16379 \
-v /docker/redis-cluster/redis1/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis1/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
--------------------------------------------------------------------------
docker run -d \
--name redis2 \
--net=host \
-v /docker/redis-cluster/redis2/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis2/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
--------------------------------------------------------------------------
docker run -d \
--name redis3 \
--net=host \
-v /docker/redis-cluster/redis3/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis3/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
--------------------------------------------------------------------------
docker run -d \
--name redis4 \
--net=host \
-v /docker/redis-cluster/redis4/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis4/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
--------------------------------------------------------------------------
docker run -d \
--name redis5 \
--net=host \
-v /docker/redis-cluster/redis5/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis5/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes
--------------------------------------------------------------------------
docker run -d \
--name redis6 \
--net=host \
-v /docker/redis-cluster/redis6/conf/redis.conf:/etc/redis/redis.conf \
-v /docker/redis-cluster/redis6/data:/data \
redis redis-server /etc/redis/redis.conf --appendonly yes


# 使用redis-cli一键搭建集群
/root/test/redis-5.0.0/src/redis-cli --cluster create 39.108.158.33:16371 39.108.158.33:16372 39.108.158.33:16373 140.143.0.227:16371 140.143.0.227:16372 140.143.0.227:16373 --cluster-replicas 1 -a agefades

# -a    密码

# 访问时使用redis集群访问
/root/redis-5.0.0/src/redis-cli -h 127.0.0.1 -p 16374  -c

```

```shell
# 公网搭建注意事项
# 如果公网搭建一直处于连接状态，那么需要去公网哪台redis将其他节点的公网ip配置完成
cluster meet 39.108.158.33 16371
cluster meet 39.108.158.33 16372
cluster meet 39.108.158.33 16373
cluster meet 140.143.0.227 16371
cluster meet 140.143.0.227 16372
cluster meet 140.143.0.227 16373
```

