[TOC]

# 诸葛 - Redis

## Redis 核心数据结构与高性能原理

### Redis 安装

```shell
# 安装地址
https://redis.io/
```

```shell
# 安装步骤

# 安装 gcc
yum install gcc

# 下载、解压、编译
wget http://download.redis.io/releases/redis-5.0.8.tar.gz

tar xzf redis-5.0.8.tar.gz

cd redis-5.0.8

make

  # 启动并指定配置文件，后台启动修改 redis.conf 里的 daemonize 改为 yes
src/redis-server redis.conf

# 验证启动是否成功
ps -ef | grep redis

# 进入 redis 客户端
src/redis-cli

# 退出客户端
quit

# 退出 redis 服务
	# pkill redis-server
	# kill 进程号
	# src/redis-cli shutdown
```

### Redis 目录结构

![UTOOLS1585706867490.png](http://yanxuan.nosdn.127.net/461431da6ff37aa5d13044367f6d3c79.png)

### Redis 版本简述

```shell
# Redis6.0 版本将支持多线程，理论上可以达到 10w qps/s 的并发。
```

### Redis 常用数据结构

![UTOOLS1585707053806.png](http://yanxuan.nosdn.127.net/5363e3932d019f03b639221df533fcd1.png)

#### String

```shell
# 字符串常用操作
set key value # 存入字符串键值对

mset key value [key value...] # 批量存储字符串键值对

setnx key value # 存入一个不存在的字符串键值对

get key # 获取一个字符串键值

mget key [key...] # 批量获取字符串键值

del key [key...] # 删除键

expire key seconds # 设置一个键的过期时间（秒）

# 原子加减
incr key # 将 key 中储存的数字加1

decr key # 将 key 中储存的数字减1

incrby key increment # 将 key 中储存的数字加 increment

decrby key decrement # 将 key 中所储存的值减 decrement
```

##### String 应用场景

```shell
# 单值缓存
set key value
get key

# 对象缓存
set user:1 value(json 数据)
get user:1

# 分布式锁
setnx product:1001 true # 返回 1 代表获取锁成功
setnx product:1001 true # 返回 0 代表获取锁失败
	# ... 执行业务操作
del product:1001 # 执行完业务释放锁
set product:1001 true ex 10 nx # 防止程序意外终止导致死锁

# 计数器
incr article:readcount:1001
get article:readcount:1001

# Web 集群 session 共享
spring session + redis 实现 session 共享

# 分布式系统全局序列号
incrby orderId 1000 # redis 批量生成序列号提升性能
```

#### Hash

```shell
# Hash 常用操作
hset key field value # 存储一个哈希表 key 的键值

hsetnx key field value # 存储一个不存在的哈希表key 的键值

hmset key field value [field value...] # 在一个哈希表key 中存储多个键值对

hget key field # 获取哈希表中key 对应的field 键值

hmget key field [field...] # 批量获取哈希表中 key 多个 field 键值

hdel key field [field...] # 删除哈希表中 key 的 field 键值

hlen key # 返回哈希表中 key 中 field 的数量

hgetall key # 返回哈希表中所有的键值

hincrby key field increment # 为哈希表中 key 中 field 的值自增 increment
```

##### Hash 应用场景

```shell
# 对象缓存
hmset user 1:name swordsman 1:balance 1888
hmget user 1:name 1:balance
```

```shell
# 电商购物车
	# 以用户 id 为 key
	# 商品 id 为 field
	# 商品数量为 value
	
# 购物车操作
	# 添加商品: hset cart:1001 10088 1
	# 增加数量: hincrby cart:1001 10088 1
	# 商品总数: hlen cart:1001
	# 删除商品: hdel cart:1001 10088
	# 获取购物车所有商品: hgetall cart:1001
```

![UTOOLS1585792710631.png](http://yanxuan.nosdn.127.net/5ccd48be35475b9319ee57891c6f5e09.png)

##### Hash 优缺点

```shell
# 优点:
	# 同类数据归类整合储存，方便数据管理
	# 相比 string 操作消耗内存和 cpu 更小
	# 相比 string 储存更节省空间
	
# 缺点:
	# 过期功能不能使用在 field 上，只能用在 key 上
	# redis 集群架构下不适合大规模使用
```

#### List

```shell
# List 常用操作
lpush key value [value...] # 将一个或多个 value 插入到key 列表的表头（最左边）

rpush key value [value...] # 将一个或多个 value 插入到key 列表的表尾（最右边）

lpop key # 移除并返回 key 列表的头元素

rpop key # 移除并返回 key 列表的尾元素

lrange key start stop # 返回列表 key 中指定区间内的元素，start - stop

blpop key [key...] timeout # 从表头弹出一个元素，若没有，则阻塞等待 timeout（0 则一直等待）

brpop key [key...] timeout # 从表尾弹出一个元素，若没有，则阻塞等待 timeout（0 则一直等待）
```

![UTOOLS1585793531096.png](http://yanxuan.nosdn.127.net/0fef4b8e3417a2bbffc3911ffc1e9809.png)

##### List 常用数据结构

```shell
# 栈 = lpush + lpop -> FILO

# 队列 = lpush + rpop

# 阻塞队列 = lpush + brpop
```

##### List 应用场景

```shell
# 微博和微信公号消息流
```

![UTOOLS1585879682541.png](http://yanxuan.nosdn.127.net/9e678eade990961ee8875648a36cfac4.png)

#### Set

```shell
# Set 常用操作
sadd key member [member...] # 往集合 key 中存入元素，元素存在则忽略

srem key member [member...] # 从集合 key 中删除元素

smembers key # 获取集合 key 中所有元素

scard key # 获取集合 key 的元素个数

sismember key member # 判断 member 元素是否存在集合 key 中

srandmember key [count] # 从集合 key 中随机选出 count 个元素，元素不从 key 中删除

spop key [count] # 从集合 key 中随机选出 count 个元素，元素从 key 中删除

# Set 运算操作
sinter key [key...] # 交集运算

sinterstore destination key [key...] # 将交集结果存入新集合 destination 中

sunion key [key...] # 并集运算

sunionstore destination key [key...] # 将并集结果存入新集合 destination 中

sdiff key [key...] # 差集运算（以第一个集合为基准）

sdiffstore destination key [key...] # 将差集结果存入新集合 destination 中
```

##### Set 应用场景

![UTOOLS1585881270506.png](http://yanxuan.nosdn.127.net/e5cb19455434a0d2b144cd938ab9a67f.png)

![UTOOLS1585881575012.png](http://yanxuan.nosdn.127.net/8d61c2292de026cdea411271eaf68b0f.png)

![UTOOLS1585884259123.png](http://yanxuan.nosdn.127.net/e8358864c862ae3d151dc89396c7449f.png)

![UTOOLS1585885092022.png](http://yanxuan.nosdn.127.net/0a4a5f5237aa5641df4e0e0d82c41210.png)

#### Zset

```shell
# Zset 常用操作
zadd key score member [[score member]...] # 往有序集合 key 中加入带分值元素

zrem key member [member ...] # 从有序集合 key 中删除元素

zscore key member # 返回有序集合 key 中元素 member 的分值

zincrby key increment member # 为有序集合 key 中元素 member 的分值加上 increment

zcard key # 返回有序集合 key 中元素个数

zrange key start stop [withscores] # 正序获取有序集合 key 从 start 下标到 stop 下标的元素

zrevrange key start stop [withscores] # 倒叙获取有序集合 key 从 start 下标到 stop 下标的元素

# Zset 集合操作
zunionstore destkey numkeys key [key...] # 并集计算
zinterstore destkey numkeys key [key...] # 交集运算
```

![UTOOLS1585885748094.png](http://yanxuan.nosdn.127.net/77c9a389b3370d2c19078829ce9140c8.png)

##### Zset 应用场景

![UTOOLS1585899420677.png](http://yanxuan.nosdn.127.net/a871ac412f2acfb403ff92c9f897e7a6.png)

### Redis 更多应用场景

```shell
# 微博、微信、陌陌（附近的人）

# 微信（摇一摇、抢红包）

# 滴滴打车、摩拜单车（附近的车）

# 美团和饿了么（附近的餐馆）

# 搜索自动补全

# 布隆过滤器

# ....
```

### Redis 的单线程和高性能

```shell
# Redis 单线程为什么还能这么快？
	# 因为所有数据都在内存中，运算都是内存级别的运算。
	
	# 单线程避免了多线程切换的开销。
	
	# 单线程要小心使用耗时操作，	比如 keys，一不小心就可能导致 redis 卡顿。
```

```shell
# redis 单线程如何处理那么多的并发客户端连接？
	# redis 利用 epoll 来实现 IO 多路复用。
	
	# 将连接信息和事件放到队列中，依次放到文件事件分配器，事件分配器将事件分发给事件处理器。
	
	# nginx 也是采用 IO 多路复用原理解决 c10k 问题。
```

![UTOOLS1585900432461.png](http://yanxuan.nosdn.127.net/4b4cda14b8c58e654d62f295555b3188.png)

```shell
# 查看 redis 支持的最大连接数，在 redis.conf 文件中可以修改
config get maxclients
	# "maxclients"
	# "10000"
	# redis 默认支持最大 1W 的连接数同时请求，将请求任务置入内部排序后由单线程处理
```

### 其他高级命令

```shell
# keys: 全量遍历键，用来列出所有满足特定正则字符串规则的 key。
	# 当 redis 数据量较大时，性能比较差，要避免使用
```

![UTOOLS1586223569949.png](http://yanxuan.nosdn.127.net/85a5540c77163f556de734889a843821.png)

```shell
# scan: 渐进式遍历键
scan cursor [match pattern] [count count]
	# cursor: 游标
	# [match pattern]: key 的正则模式
	# [count count]: 一次遍历的 key 的数量，并不是符合条件的结果数量
```

![UTOOLS1586223879363.png](http://yanxuan.nosdn.127.net/a7f77a55411e5a214df3fc7338780c6e.png)

```shell
# info: 查看 redis 服务运行信息
	# Server: 服务器运行的环境参数
	# Clients: 客户端相关信息
	# Memory: 服务器运行内存统计数据
	# Persistence: 持久化信息
	# Stats: 通用统计数据
	# Replication: 主从复制相关信息
	# CPU: CPU 使用情况
	# Cluster: 集群信息
	# KeySpace: 键值对统计数量信息
```

![UTOOLS1586224198853.png](http://yanxuan.nosdn.127.net/23ab220427e014adfb64d3bc9fe4253d.png)

### Redis 危险命令禁用 keys、flushdb、flushall、config 及解决方案

```shell
# keys:
	# 数据量大时可能导致 redis 锁住及 cpu 飙升
	# redis 服务卡住可能导致系统雪崩！
	
# flushdb:
	# 删除 redis 中当前所在数据库中的所有记录
	
# flushall:
	# 删除 redis 中所有数据库中的所有记录
	
# config:
	# 客户端可修改 redis 配置
```

![UTOOLS1586223420064.png](http://yanxuan.nosdn.127.net/4ee126008de0de62b608674aab5ff606.png)

```shell
# 禁用命令
rename-command KEYS     ""
rename-command FLUSHALL ""
rename-command FLUSHDB  ""
rename-command CONFIG   ""
```

```shell
# 重命名命令
rename-command KEYS     "XXXXX"
rename-command FLUSHALL "XXXXX"
rename-command FLUSHDB  "XXXXX"
rename-command CONFIG   "XXXXX"
```

## Redis 持久化、主从与哨兵架构详解

### Redis 持久化

#### RDB 快照

```shell
# 默认情况下，redis 将内存数据库快照保存在名字为 dump.rdb 的二进制文件中。

# 可以对 redis 进行设置，在 N 秒内数据集至少有 M 个改动，这一条件被满足时，自动保存一次数据集。

# 如果想关闭 rdb 只需将 redis.conf 里所有的 save 保存策略注释掉即可。
```

![UTOOLS1586226633947.png](http://yanxuan.nosdn.127.net/76ff568a77825e2e1ab3bdc4dbf7a3a2.png)

```shell
# 可以手动执行命令生成 rdb 快照:
	# 进入客户端执行 save 或 bgsave 即可生成 dump.rdb 文件。
	
	# 每次命令执行都会将所有 redis 内存快照到一个新的 rdb 文件里，并覆盖原有 rdb 快照文件。
```

```shell
# save: 同步命令

# bgsave: 异步命令
	# bgsave 会从 redis 主进程 fork 出一个子进程专门用来生成 rdb 快照文件
	
	# fork() 是 linux 函数
```

##### save 与 bgsave 对比

| 命令                    | save             | bgsave                              |
| ----------------------- | ---------------- | ----------------------------------- |
| IO类型                  | 同步             | 异步                                |
| 是否阻塞 redis 其他命令 | 是               | 否（fork 生成子进程时会有短暂阻塞） |
| 复杂度                  | O(n)             | O(n)                                |
| 优点                    | 不会消耗额外内存 | 不阻塞客户端命令                    |
| 缺点                    | 阻塞客户端命令   | 消耗额外内存                        |

```shell
# redis.conf 里自动生成 rdb 文件后台使用的是 bgsave 方式
```

#### AOF

```shell
# 突然宕机时，rdb 持久化可能丢失部分还未同步到 dump.rdb 文件中的数据。

# 1.1 版本开始，redis 支持 AOF 持久化。
	# 将对数据修改的每一条指令记录进文件 appendonly.aof 中。
	
# redis.conf 里修改配置以打开 AOF 功能。
	# appendonly yes
	
# 此时 redis 如果宕机重启，将会重新执行 AOF 文件中的命令来达到重建数据集的目的。
```

![UTOOLS1586227283621.png](http://yanxuan.nosdn.127.net/cd09a7ef6eaa38d0d4c1a8e8ce4e8f39.png)

##### appendfsync

```shell
# 通过配置 appendfsync 来指定多久才将数据 fsync 到磁盘一次。
	
	# appendfsync always:
		# 每次有新命令追加到 AOF 文件时就执行一次 fsync，性能非常慢、也非常安全。
	
	# appendfsync everysec:
		# 每秒 fsync 一次，性能和使用 rdb 持久化差不多，并且在故障时只会丢失 1s 内的数据。
	
	# appendsync no:
		# 从不 fsync，将数据交给 OS 处理。性能更好、但数据安全系数较低。
	
# 推荐（默认）的策略为 everysec，兼顾性能与安全。
```

![UTOOLS1586227243648.png](http://yanxuan.nosdn.127.net/c4ddc9ba06ef50d51b9b4751014dd0cd.png)

##### AOF 重写

```shell
# AOF 持久化文件中，可能存在太多没有的指令，所以 AOF 会定期根据 内存的最新数据 重写 AOF 文件。
```

![UTOOLS1586228651975.png](http://yanxuan.nosdn.127.net/0949065a5a881a102db7728ac7abbc62.png)

```shell
# AOF 自动重写的配置:

	# auto-aof-rewrite-min-size 64mb: AOF 文件至少达到 64M 才会自动重写。
	
	# auto-aof-rewrite-percentage 100: AOF 文件自上一次重写后文件大小增长了 100% 则重写。
	
# AOF 手动重写:
	
	# 进入 redis 客户端执行命令 bgrewriteaof 重写 AOF
	
# AOF 重写时，redis 会fork 出一个子进程去做，不会对 redis 正常命令处理有太多影响。
```

##### RDB 和 AOF 的比较

| 命令       | RDB        | AOF          |
| ---------- | ---------- | ------------ |
| 启动优先级 | 低         | 高           |
| 体积       | 小         | 大           |
| 恢复速度   | 快         | 慢           |
| 数据安全性 | 容易丢数据 | 根据策略决定 |

```shell
# redis 启动时，如果 rdb 和 aof 都有，优先选择 aof 文件恢复数据。
```

#### Redis 4.0 混合持久化

```shell
# redis 重启时，很少使用 rdb 恢复内存数据，因为可能导致大量数据丢失。

# 通常使用 aof 恢复数据，但是性能比 rdb 慢很多。

# 当 redis 数据很大时，启动通过 aof 恢复数据可能要花很长的时间。

# redis4.0 通过 混合持久化 解决上述问题。
```

```shell
# redis.conf 修改配置开启混合持久化:
aof-use-rdb-preamble yes
```

```shell
# 开启了混合持久化后，AOF 在重写时，不再是单纯将内存数据转换为 RESP 命令写入 AOF 文件，
	
	# 而是将重写这一刻之前的内存数据做 rdb 快照处理，并将 rdb 和增量的 aof 修改数据命令存在一起，
	
	# 写入新的 aof 文件，该文件开始不叫 appendonly.aof，需要等到重写完成后，才会改名覆盖原有文件。
	
# 于是 redis 重启之后，先加载 rdb，再加载增量 aof 日志即可完全替代之前的 aof 全量文件加载,
	
	# redis 实例重启加载数据性能大大提升。
```

![UTOOLS1586231037514.png](http://yanxuan.nosdn.127.net/e3e5fc1307aec3e52dee6cd9cc1d7a48.png)

### Redis 主从架构

![UTOOLS1586416734190.png](http://yanxuan.nosdn.127.net/ede7170a3e63abe7e07ccf4927b967c4.png)

#### 主从工作原理

```shell
# 如果 master 配置了 slave，不论 slave 是否第一次连接 master，
	# 都会发送 sync 命令(redis2.8 之前的命令) 给 master 请求复制数据。
	
# master 收到 sync 命令后，在后台执行 bgsave 生成最新 rdb 文件，
	# 在该持久化期间，master 会继续接收客户端请求，将可能修改数据集的请求缓存在内存中。
	
# rdb 生成完后，发送文件数据集给 slave，slave 把接收到的数据持久化生成 rdb，再加载到内存中，
	# master 再将之前缓存在内存中的命令发送给 slave。

# 当 master 与 slave 断开连接时，slave 能够自动重连 master,
	# 如果 master 收到了多个 slave 并发连接请求，只会进行一次持久化，而不是一个连接一次，
	# 最后再把这一份持久化的数据发送给多个并发连接的 slave。
```

```shell
# 当 master 和 slave 断开重连后，一般都会对整份数据进行复制，
	# 但从 redis2.8 之后，master 和 slave 断开重连后支持部分复制。
```

##### 部分复制

```shell
# master 会在其内存中创建一个复制数据用的缓存队列，缓存最近一段时间的数据，
	# master 和它所有的 salve 都维护了复制的数据下标 offset 和 master 的进程id，
	# 当网络连接断开后，slave 会请求 master 继续进行未完成的复制，从记录的数据下标开始。
	
# 如果 master 进程 id 变了，或者从节点数据下标 offset 太旧，已经不在 master 的缓存队列了，
	# 将会进行一次全量数据的复制。
	
# 2.8版本之后，redis 改用可以支持部分数据复制的 psync 去 master 同步数据。
```

![UTOOLS1586916289557.png](http://yanxuan.nosdn.127.net/efe25af71405611fa8b64dd9ff54e38c.png)

![UTOOLS1586916341117.png](http://yanxuan.nosdn.127.net/c5f79d0dd0814369e7cb60ebd809c376.png)

#### 主从架构搭建

```shell
https://www.jianshu.com/p/f0e042b95249
```

### Redis 哨兵架构

![UTOOLS1589868557148.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589868557148.png)

```shell
# sentinel(哨兵) 是特殊的 redis 服务，
	# 不提供读写服务，
	# 主要用来监控 redis 实例节点
	
# 哨兵架构下，client 端第一次从哨兵找出 redis 的主节点，
	# 后续就直接访问 redis 的主节点，不会每次都通过 sentinel 代理访问 redis 的主节点，
	# 当 redis 的主节点发生变化，哨兵会第一时间感知到，并且将新的主节点通知给 client 端
	# 这里面 redis 的 client 端一般都实现了订阅功能，订阅 sentinel 发布的节点变动消息
```

#### 哨兵架构搭建步骤

```shell
#1. 复制一份 sentinel.conf 文件
cp sentinel.conf sentinel-26379.conf

#2. 修改配置
vim sentinel-26379.conf

-----------------------
# sentinel 端口号
port 26379

# 是否后台启动
daemonize yes

# 进程文件地址
pidfile "/var/run/redis-sentinel-26379.pid"

# 日志文件地址
logfile "26379.log"

# 数据文件地址
dir "/usr/local/redis-5.0.3/data"

# sentinel monitor <master-name> <ip> <redis-port> <quorum>
	# quorum : 指明当有多少个 sentinel 认为一个 master 失效时，master 真正失效
		# 值一般为: sentinel总数/2 + 1
sentinel monitor redis-master 127.0.0.1 6379 2
-----------------------

#3. 启动 sentinel 哨兵实例
src/redis-sentinel sentinel-26379.conf

#4. 连接 sentinel,查看 sentinel 的 info 信息
src/redis-cli -p 26379
info

# 从 sentinel 的 info 里可以看出已经识别了 redis 的主从

# 可以再多配置几个 sentinel，修改上述配置即可
```

#### 哨兵的 Jedis 连接代码

```java
public class JedisSentinelTest {
  
  public static void main(String[] args) throws IOException {
    JedisPoolConfig config = new JedisPoolConfig();
    config.setMaxTotal(20);	// 最大线程数
		config.setMaxIdle(10);	// 最大空闲线程数
    config.setMinIdle(5);		// 最小空闲线程数
    
    String masterName = "redis-master";
    Set<String> sentinels = new HashSet<>();
    sentinels.add(new HostAndPort("127.0.0.1", 26379).toString());
    sentinels.add(new HostAndPort("127.0.0.1", 26380).toString());
    sentinels.add(new HostAndPort("127.0.0.1", 26381).toString());
    
    // JedisSentinelPool 其实本质跟 JedisPool 类似，都是与 redis 主节点建立的连接池
    // JedisSentinelPool 并不是与 sentinel 建立的连接池，
    // 而是通过 sentinel 发现的 redis 主节点并与其建立连接。
    // params: 
			// 1. 集群名称
			// 2. 哨兵集合
    	// 3. 连接池配置
			// 4. 连接超时时间
    	// 5. 连接密码
    JedisSentinelPool jedisSentinelPool = 
      			new JedisSentinelPool(masterName, sentinels, config, 3000, null);
    
    Jedis jedis = null;
    try {
      jedis = jedisSentinelPool.getResource();
      System.out.println(jedis.set("sentinel", "test"));
      System.out.println(jedis.get("sentinel"));
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      if (jedis != null) {
        jedis.close();
      }
    }
  }
  
}
```

#### SpringData Redis 连接代码

```xml
<dependency> 
  <groupId>org.springframework.boot</groupId> 
  <artifactId>spring‐boot‐starter‐data‐redis</artifactId>
</dependency>

<dependency> 
  <groupId>org.apache.commons</groupId> 
  <artifactId>commons‐pool2</artifactId>
</dependency>
```

```yaml
server:
	port: 8080
	
spring:
	redis:
    # 使用的 redis 库
    database: 0
    # 超时时间
    timeout: 3000
    # lettuce 连接池配置
    lettuce:
      pool:
        # 最大闲置线程数
        max-idle: 50
        # 最小闲置线程数
        min-idle: 10
        # 最大活跃数
        max-active: 100
        # 最大等待时间
        max-wait: 1000
    # 哨兵模式
    sentinel:
      # 主服务器所在集群名称
      master: redis-master
      # 集群节点 Host + Port
      nodes: 127.0.0.1:26379,127.0.0.1:26380,127.0.0.1:26381
```

```java
@RestController
@Slf4j
@AllArgsConstructor
public class IndexController {
  
  private final StringRedisTemplate stringRedisTemplate;
  
  /**
   * 测试节点挂了，哨兵重新选举新的 master 节点，客户端是否能动态感知到
   *   新的 master 选举出来后，哨兵会把消息发布出去，客户端实际上是实现了一个消息监听机制，
   * 	 当哨兵把新 master 的消息发布出去，客户端会立即感知到新 master 的信息，
   *	 从而动态切换访问的 master ip + port
   * 此程序运行起来后:
   *	kill 掉 6379 redis master
   * 	Java 程序尝试重连，每次重连时间为配置的 spring.redis.timeout，重连失败抛出异常并继续重连
	 * 	当 Sentinel 将某个从节点切换为主节点后，Java 程序收到通知，重连成功，继续执行程序。
	 *		Sentinel 会将选举出的新的 master 节点里的 replicaof 配置清空并重启节点
	 *		更改其他从节点的配置文件并重启
	 *	Sentinel 的配置文件也会更新一些新的东西。
	 *
	 * 如果 6379 重启成功，会成为刚才选举出来的 master 节点的从节点。
	 *	其实也是 sentinel 修改了它的配置文件
   */
  @GetMapping("/test/sentinel")
  public void testSentinel() throws InterruptedException {
    int i = 1;
    while (true) {
      try {
        stringRedisTemplate.opsForValue().set("AgeFades:" + i, i + "");
        log.info("设置的 key value 为: {} {}", "AgeFades:" + i, i + "");
        i++;
        Thread.sleep(1000);
      } catch (Exception e) {
        log.error("test/sentinel 异常 : {}", e);
      }
    }
    
  }
  
}
```

### StringRedisTemplate 与 RedisTemplate

```shell
# Spring 封装了 RedisTemplate 对象，来进行对 redis 的各种操作。
```

```shell
# RedisTemplate 中定义了对 5 种数据结构的操作。
```

```java
// 操作字符串
redisTemplate.opsForValue(); 

// 操作 hash
redisTemplate.opsForHash();

// 操作 list
redisTemplate.opsForList();

// 操作 set
redisTemplate.opsForSet();

// 操作有序 set
redisTemplate.opsForZSet();
```

```shell
# StringRedisTemplate 继承自 RedisTemplate。

# StringRedisTemplate 默认采用 String 序列化策略。

# RedisTemplate 默认采用 JDK 的序列化策略。

# 自我总结一下: 
	# StringRedisTemplate 存 String，取出来再转换（可能是 int、double、jsonObj....）
	
	# RedisTemplate 存 Object，拿出来直接自定义类型接收，但是可能类型转换失败
```

#### Redis 客户端命令对应的 RedisTemplate 方法列表

##### String

| Redis                  | RedisTemplate（接下来简写为 rt）           |
| ---------------------- | ------------------------------------------ |
| set key value          | rt.opsForValue().set("key", "value");      |
| get key                | rt.opsForValue().get("key");               |
| del key                | rt.delete("key");                          |
| strlen key             | rt.opsForValue().size("key");              |
| getset key value       | rt.opsForValue().getAndSet("key","value"); |
| getrange key start end | rt.opsForValue().get("key",start,end);     |
| append key value       | rt.opsForValue().append("key","value");    |

##### Hash

| hget key field                           | rt.opsForHash().get("key","field")                           |
| ---------------------------------------- | ------------------------------------------------------------ |
| hmset key field1 value1 field2 value2... | rt.opsForHash().putAll("key",map);<br />// map 是一个集合对象 |
| hset key field value                     | rt.opsForHash().put("key","field","value");                  |
| hexists key field                        | rt.opsForHash().hasKey("key","field");                       |
| hgetall key                              | rt.opsForHash().entries("key") <br />// 返回Map对象          |
| hvals key                                | rt.opsForHash().values("key") <br />// 返回List对象          |
| hkeys key                                | rt.opsForHash().keys("key")<br />// 返回List对象             |
| hmget key field1 field2...               | rt.opsForHash().multiGet("key",keyList)                      |
| hsetnx key field value                   | rt.opsForHash().putIfAbsent("key","field","value")           |
| hdel key field1 field2                   | rt.opsForHash().delete("key","field1","field2")              |

##### List

| lpush list node1 node2 node3... | rt.opsForList().leftPush("list","node")<br />rt.opsForList().leftPushAll("list",list) <br />// list是集合对象 |
| ------------------------------- | ------------------------------------------------------------ |
| rpush list node1 node2 node3... | rt.opsForList().rightPush("list","node") <br />rt.opsForList().rightPushAll("list",list)<br />// list是集合对象 |
| lindex key index                | rt.opsForList().index("list", index)                         |
| llen key                        | rt.opsForList().size("key")                                  |
| lpop key                        | rt.opsForList().leftPop("key")                               |
| rpop key                        | rt.opsForList().rightPop("key")                              |
| lpushx list node                | rt.opsForList().leftPushIfPresent("list","node")             |
| rpushx list node                | rt.opsForList().rightPushIfPresent("list","node ")           |
| lrange list start end           | rt.opsForList().range("list",start,end)                      |
| lrem list count value           | rt.opsForList().remove("list",count,"value")                 |
| lset key index value            | rt.opsForList().set("list",index,"value")                    |

##### Set

| sadd key member1 member2... | rt.boundSetOps("key").add("member1","member2",...)<br/>rt.opsForSet().add("key", set) <br />// set是一个集合对象 |
| --------------------------- | ------------------------------------------------------------ |
| scard key                   | rt.opsForSet().size("key")                                   |
| sdiff key1 key2             | rt.opsForSet().difference("key1","key2") <br />// 返回一个集合对象 |
| sinter key1 key2            | rt.opsForSet().intersect("key1","key2")<br />// 同上         |
| sunion key1 key2            | rt.opsForSet().union("key1","key2")<br />// 同上             |
| sdiffstore des key1 key2    | rt.opsForSet().differenceAndStore("key1","key2","des")       |
| sinter des key1 key2        | rt.opsForSet().intersectAndStore("key1","key2 ","des")       |
| sunionstore des key1 key2   | rt.opsForSet().unionAndStore("key1","key2"," des")           |
| sismember key member        | rt.opsForSet().isMember("key","member")                      |
| smembers key                | rt.opsForSet().members("key")                                |
| spop key                    | rt.opsForSet().pop("key")                                    |
| srandmember key count       | rt.opsForSet().randomMember("key",count)                     |
| srem key member1 member2... | rt.opsForSet().remove("key","member1","member2",...)         |

