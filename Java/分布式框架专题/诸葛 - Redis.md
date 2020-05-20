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
    sentinels.add(new HostAndPort("127.0.0.1", 26379).toString());
    sentinels.add(new HostAndPort("127.0.0.1", 26379).toString());
    
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

## Redis 缓存高可用集群

### Redis 集群方案比较

![UTOOLS1589875543782.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589875543782.png)

```shell
# 在 redis3.0 以前的版本要实现集群一般是借助哨兵 sentinel 工具来监控 master 节点的状态。
	# 如果 master 节点异常，则会做主从切换，将某一台 slave 作为 master。
	# 哨兵的配置比较复杂，并且性能和高可用性等各方面表现一般，
	# 特别是在主从切换的瞬间存在 访问瞬断 的情况，
	# 而且哨兵模式只有一个主节点对外提供服务，没法支持很高的并发，
	# 且单个主节点内存也不宜设置的过大，否则会导致持久化文件过大，影响数据恢复或主从同步的效率。
```

![UTOOLS1589879078806.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589879078806.png)

```shell
# redis 集群是一个由多个 主从节点群 组成的分布式服务集群，
	# 具有 复制、高可用、分片 等特性。
	
# redis 集群不需要 sentinel 哨兵也能完成节点移除和故障转移的功能。

# 需要将每个节点设置成集群模式，这种集群模式没有中心节点，可以水平扩展，
	# 官方称: 可以线性扩展到上万个节点（推荐不超过 1000 个节点）
	
# redis 集群的性能和高可用性均优于之前版本的哨兵模式，且集群配置非常简单。
```

### Redis 集群搭建

```shell
# redis 集群至少需要三个 master 节点，最好每个 master 再配置一个 slave 节点

# 一般 redis 集群都是 6个节点起步，三台机器，每台机器一主一从（交叉主从）
```

```shell
# 先按此文档最开始的操作安装 redis

# 在第一台机器的 /usr/local 下创建文件夹 redis-cluster
mkdir -p /usr/local/redis-cluster

# 创建 redis prot 文件夹
mkdir 8001 8004

# 把之前的 redis.conf 复制到 8001 下，修改如下内容
-------------------------
daemonize yes
port 8001 # 分别对每个机器的端口号进行设置
dir /usr/local/redis-cluster/8001/ # 指定数据文件存放的位置，必须要指定不同的目录，不然会丢失数据
cluster-enabled yes # 启动集群模式
cluster-config-file nodes-8001.conf # 集群节点信息文件，800x 最好和 port 对应上
cluster-node-timeout 5000 # 集群节点连接超时时间
# bind 127.0.0.1 (去掉 bind 绑定访问 ip 信息)
protected-mode no # 关闭保护模式
appendonly yes
# 如果需要设置密码需要增加如下配置:
requirepass swordsman # 设置 redis 访问密码
masterauth swordsman # 设置集群节点间访问密码，跟上面一致
-------------------------

# 把修改后的配置文件，copy 到8004，修改第 2、3、5 行里的端口号即可
# 可以用批量替换: :%s/源字符串/目标字符串/g

# 另外两台机器也需要做上面几步操作，第二台机器用8002和8005，第三台用8003和8006

# 分别启动 6 个 redis 实例，然后检查是否启动成功
${redis_home}/src/redis-server /usr/local/redis-cluster/800*/redis.conf
ps -ef | grep redis

# 用 redis-cli 创建整个 redis 集群（redis 5 之前的版本都是用 ruby 脚本实现）

# 执行这条命令需要确认三台机器之间的 redis 实例要能相互访问，虚拟机就是关闭防火墙，服务器就是开安全组
# 关闭防火墙:
systemctl stop firewalld
systemctl disable firewalld

# 创建集群命令:
# 下面命令里的 1 代表为每个创建的主服务器节点创建一个从服务器节点
# ${host:post}... 是6个实例的 ip 加端口
${redis_home}/src/redis-cli -a swordsman --cluster create --cluster-replicas 1 ${host:port}...

# 验证集群
# 连接任意一个客户端即可
# -a 密码 -c 集群模式 -h ip -p port
${redis_home}/src/redis-cli -a swordsman -c -h ${host} -p 800*

# 进行验证
cluster info # 查看集群信息
cluster nodes # 查看节点列表

# 进行数据操作验证

# 关闭集群则需要逐个进行关闭，使用命令
${redis_home}/src/redis-cli -a swordsman -c -h ${host} -p 800* shutdown
```

### Redis 集群原理分析

```shell
# Redis Cluster 将所有数据划分为 16384 个 slots(哈希槽)，
	# 每个节点负责其中一部分槽位，槽位的信息存储于每个节点中。
	
# 当 Redis Cluster 的客户端来连接集群时，它也会得到一份集群的槽位配置信息并将其缓存在客户端本地，
	# 当客户端要查找某个 key 时，可以直接定位到目标节点，
	# 同时，因为槽位的信息可能会存在客户端与服务器不一致的情况，还需要纠正机制来实现槽位信息的校验调整。
```

#### 槽位定位算法

```shell
# Cluster 默认会对 key 值使用 crc16 算法进行 hash 得到一个整数值，
	# 然后用这个整数值对 16384 进行取模来得到具体槽位。

HASH_SLOT = CRC16(key) mod 16384
```

#### 跳转重定向

```shell
# 当客户端向一个错误的节点发出了指令，该节点会发现指令的 key 所在的槽位并不归自己管理，
	# 此时该节点会向客户端发送一个特殊的跳转指令携带目标操作的节点地址，
	# 告诉客户端去连这个节点去获取数据，
	# 客户端收到指令后除了跳转到正确的节点上去操作，
	# 还会同步更新纠正本地的槽位映射表缓存，
	# 后续所有 key 将使用新的槽位映射表。
```

![UTOOLS1589940920121.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589940920121.png)

#### Redis 集群节点间的通信机制

```shell
# redis cluster 节点间采取 gossip 协议进行通信，

# 维护集群的元数据有两种方式: 集中式 和 gossip
```

##### 集中式

```shell
# 优点在于元数据的更新和读取，时效性非常好，
	# 一旦元数据出现变更立即就会更新到集中式的存储中，
	# 其他节点读取的时候立即就可以感知到。
	
# 缺点在于所有的元数据的更新压力全部集中在一个地方，可能导致元数据的存储压力。
```

##### gossip

```shell
# gossip 协议包含多种消息，包括 ping、pong、meet、fail 等等...

# ping:
	# 每个节点都会频繁的给其他节点发送 ping,
	# 其中包含自己的状态还有自己维护的集群元数据，
	# 互相通过 ping 交换元数据。
	
# pong:
	# 返回 ping 和 meet, 包含自己的状态和其他信息，也可以用于信息广播和更新。
	
# fail:
	# 某个节点判断另一个节点 fail 之后，就发送 fail 给其他节点，通知其他节点该节点宕机了。
	
# meet:
	# 某个节点发送 meet 给新加入的节点，让新节点加入集群中，
	# 然后新节点就会开始与其他节点进行通信，
	# 不需要发送形成网络的所需的所有 cluster meet 命令。
	# 发送 cluster meet 消息以便每个节点能够达到其他节点只需要通过一条已知的节点链就够了。
	# 由于在心跳包中会交换 gossip 信息，将会创建节点间缺失的链接。
	
# gossip 协议的优点在于: 
	# 元数据的更新比较分散，不是集中在一个地方，更新请求会陆陆续续，打到所有节点上去更新，
	# 有一定的延时，降低了压力。
	
# gossip 协议的缺点在于:
	# 元数据更新有延时，可能导致集群的一些操作会有一些滞后。
	
# 10000 端口:
	# 每个节点都有一个专门用于节点间通信的端口，就是自己提供服务的端口号 + 10000，
	# 比如 7001，那么用于节点间通信的就是 17001 端口。
	# 每个节点每隔一段时间都会往另外几个节点发送 ping 消息，
	# 同时其他节点接收到 ping 消息之后会返回 pong 消息。
```

![UTOOLS1589941226774.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589941226774.png)

##### 网络波动

```shell
# 机房网络经常发生网络波动，突然之间部分连接变得不可访问，然后很快又恢复正常。

# 为了解决这种问题，redis cluster 提供了一种选项 cluster-node-timeout，
	# 表示当某个节点持续 timeout 的时间失联时，才可以认定该节点出现故障，
	# 如果没有这个选项，网络波动会导致主从频繁切换（数据的重新复制）
```

##### Redis 集群选举原理分析

```shell
# 当 slave 发现自己的 master 变为 fail 状态时，
	# 便尝试进行 failover，以期成为的 master。
	# 由于挂掉的 master 可能会有多个 slave，从而存在多个 slave 竞争称为 master 的过程。
	
# 过程如下:
	# slave 发现自己的 master 变为 fail
	# 将自己记录的集群 currentEpoch 加1，并广播 failover_auth_request 信息
	# 其他节点收到该信息，只有 master 响应，判断请求者的合法性，
		# 并发送 failover_auth_ack，对每一个 epoch 只发送一次 ack
	# 尝试 failover 的 slave 收集 master 返回的 failover_auth_ack
	# slave 收到 超过半数master 的 ack 后，变为新 master
		# 这里解释了集群为什么至少需要三个主节点，
		# 如果只有两个，当其中一个挂了，只剩一个主节点是不能选举成功的
	# 广播 pong 消息通知其他集群节点。
	
# 从节点并不是在主节点一进入 fail 状态就马上尝试发起选举，而是有一定延迟，
	# 一定的延迟确保我们等待 fail 状态在集群中传播，
	# slave 如果立即尝试选举，其他 master 或许尚未意识到 fail 状态，可能会拒绝投票
	
# 延迟计算公式:
# slave_rank 表示此 slave 已经从 master 复制数据的总量的 rank
	# rank 越小代表已复制的数据越新。
	# 这种方式下，持有最新数据的 slave 将会首先发起选举（理论上）
DELAY = 500ms + random(0 ~ 500ms) + SLAVE_RANK * 1000ms
```

##### 集群是否完整才能对外提供服务

```shell
# 当 redis.conf 的配置 cluster-require-full-coverage 为 no 时，
	# 表示当负责一个插槽的主库下线且没有相应的从库进行故障恢复时，
	# 集群仍然可用，如果为 yes 则集群不可用。
```

##### Redis 集群为什么至少需要三个 master 节点，并且推荐节点数为奇数？

```shell
# 因为新 master 的选举需要大于半数的集群 master 节点同意才能选举成功，
	# 如果只有两个 master 节点，当其中一个挂了，是达不到选举新 master 的条件的。
	
# 奇数个 master 节点可以在满足选举该条件的基础下节省一个节点，
	# 比如三个 master 节点和四个 master 节点的集群相比，
	# 大家如果都挂了一个 master 节点都能选举新 master 节点，
	# 如果都挂了两个 master 节点都没法选举新 master 节点。
	# 所以奇数的 master 节点更多的是从节省机器资源角度出发说的。
```

##### 哨兵 lead 选举流程

```shell
# 当一个 master 服务器被某 sentinel 视为客观下线状态后，
	# 该 sentinel 会与其他 sentinel 协商选出 sentinel 的 leader 进行故障转移工作。
	# 每个发现 master 服务器进入客观下线的 sentinel，
  # 都可以要求其他 sentinel 选自己为 sentinel 的 leader，选举是先到先得。
  # 同时，每个 sentinel 每次选举都会自增配置纪元（选举周期）,
  # 每个纪元中只会选择一个 sentinel 的 leader。
  # 如果有超过一半的 sentinel 选举某 sentinel 作为 leader，
  # 之后将由该 sentinel 进行故障转移操作，从存活的 slave 中选举出新的 master,
  # 这个选举过程跟集群的 master 选举很类似。
  
# 哨兵集群只有一个哨兵节点，redis 的主从也能正常运行以及选举 master，
	# 如果 master 挂了，那唯一的那个哨兵节点就是哨兵 leader 了，
	# 可以正常选举新 master。
	
# 不过为了高可用，一般都推荐至少部署三个哨兵节点。

# 为什么推荐奇数个节点，原理跟集群奇数节点一致。
```

## Redis 高可用集群之水平扩展

```shell
# Redis3.0 以后的版本虽然有了集群功能，提供了比之前版本的哨兵模式更高的性能与可用性，
	# 但是集群的水平扩展却比较麻烦。
	
# 下面演示 redis 高可用集群如何做水平扩展，
	# 原始集群由 6 个节点组成，6个节点分布在三台机器上，采用三主三从的模式。
```

![UTOOLS1589945124538.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589945124538.png)

### 启动集群

#### 启动整个集群

```shell
${redis_home}/src/redis-server /usr/local/redis-cluster/800*/redis.conf
```

#### 客户端连接实例

```shell
${redis_home}/src/redis-cli -a swordsman -c -h ${host} -p ${port}
```

#### 查看集群状态

```shell
cluster nodes
```

![UTOOLS1589945488939.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589945488939.png)

```shell
# 从上图看出，集群运行正常，3个 master，3个 slave

# 8001 端口的实例节点存储 0-5460 这些 hash 槽

# 8002 端口的实例节点存储 5461-10922 这些 hash 槽

# 8003 端口的实例节点存储 10923-16383 这些 hash 槽

# 三个 master 节点存储的所有 hash 槽组成 redis 集群的存储槽位

# slave 是每个主节点的备份从节点，不显示存储槽位
```

### 集群操作

```shell
# 我们在原始集群基础上再增加 一主(8007) 一从(8008)

# 增加节点后的集群参见下图，新增节点用虚线框表示。
```

![UTOOLS1589952619213.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589952619213.png)

#### 增加实例

```shell
# 在 /usr/local/redis-cluster 下创建 8007 和 8008 文件夹，
	# 并拷贝 8001 文件夹下的 redis.conf 文件到 8007 和 8008 两个目录下。

# 修改配置文件
-----------------
port: ${port}
dir /usr/local/redis-cluster/${port}/
cluster-config-file nodes-${port}.conf
-----------------

# 启动服务 & 查看状态
${redis_home}/src/redis-server /usr/local/redis-cluster/800*/redis.conf
ps -ef | grep redis
```

#### redis 集群命令帮助

```shell
${redis_home}/src/redis-cli --cluster help
```

![UTOOLS1589953075729.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589953075729.png)

```shell
# create: 创建一个集群环境 host1:port1 ... hostN:portN

# call: 可以执行 redis 命令

# add-node: 添加节点至集群中，第一个参数为新节点的 ip:port，
	# 第二个参数为集群中任意一个已经存在的节点的 ip:port
	
# del-node: 移除一个节点

# reshard: 重新分片

# check: 检查集群状态
```

#### 配置 8007 为集群主节点

```shell
# 使用 add-node 命令新增一个主节点 8007(master)
${redis_home}/src/redis-cli -a swordsman --cluster add-node ip:port ip:port

# 看到日志最后有下述提示则表示新节点加入成功
"[OK] New node added correnctly"
```

```shell
# 查看集群状态
${redis_home}/src/redis-cli -a swordsman -c -h ip -p port
cluster nodes
```

![UTOOLS1589953562919.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589953562919.png)

```shell
# 当添加节点成功以后，新增的节点不会有任何数据，
	# 因为它还没有分配任何的 hash 槽，需要自己为新节点手动分配 hash 槽。
	
# 使用 redis-cli 命令为新节点分配 hash 槽，
	# 找到集群中的任意一个主节点，对其进行重新分片。
${redis_home}/src/redis-cli -a sworsman --cluster reshard ip:port
```

![UTOOLS1589954211624.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589954211624.png)

```shell
# 查看集群状态
${redis_home}/src/redis-cli -a swordsman -c -h ip -p port
cluster nodes
```

![UTOOLS1589954505831.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589954505831.png)

```shell
# 此时 8007 已经 hash 槽了，也就是说可以在 8007 上读写数据了。

# 到此为止，8007 已经加入到了集群中，并且是主节点。
```

#### 配置 8008 为 8007 的从节点

```shell
# 添加 8008 到集群中 & 查看集群状态
${redis_home}/src/redis-cli -a swordsman --cluster add-node ip:port ip:port
```

![UTOOLS1589954642284.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589954642284.png)

```shell
# 此时 8008 还是一个 master 节点，没有被分配任何的 hash 槽。

# 我们需要执行 replicate 命令来指定当前节点（从节点）的主节点 id 为哪个，
	# 首先需要连接进新加入的 8008 节点的客户端，
	# 然后使用集群命令操作，将 8008 指定为 8007 的从节点
${redis_home}/src/redis-cli -a swordsman -c -h ip -p port
cluster replicate xxx(主节点的 id)
```

```shell
# 查看集群状态
```

![UTOOLS1589955513241.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589955513241.png)

#### 删除 8008 从节点

```shell
# 用 del-node 删除 8008 从节点
${redis_home}/src/redis-cli -a swordsman --cluster del-node ip:port xxx(节点id)
```

```shell
# 查看集群状态，8008 已经移除，并且该节点的 redis 服务也已经被停止
```

![UTOOLS1589955663217.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589955663217.png)

#### 删除 8007 主节点

```shell
# 删除主节点的步骤会比较麻烦，因为主节点里面分配了 hash 槽，
	# 所以必须先把该节点的 hash 槽放入到其他的可用主节点上，再进行移除操作，
	# 否则会出现数据丢失问题。
	# 目前只能把 master 的数据迁移到一个节点上，暂时做不了平均分配功能。
${redis_home}/src/redis-cli -a swordsman --cluster reshard ip:port
```

![UTOOLS1589955799233.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589955799233.png)

```shell
# 至此，我们已经成功的把 8007 主节点的数据迁移到 8001 上去了，
	# 现在可以查看集群状态，8007 节点已经没有任何 hash 槽了，证明迁移成功。
```

![UTOOLS1589955868163.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589955868163.png)

```shell
# 最后直接使用 del-node 命令删除 8007 主节点即可。
${redis_home}/src/redis-cli -a swordsman --cluster del-node ip:port xxx(节点id)
```

```shell
# 最后查看集群状态，一切都还原为最初始的状态了。
```

![UTOOLS1589955938813.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589955938813.png)

## Redis 缓存设计与性能优化

![UTOOLS1589956047183.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589956047183.png)

### 缓存设计

#### 缓存穿透

```shell
# 指查询一个根本不存在的数据，缓存层和存储层都不会命中，
	# 通常出于容错的考虑，如果从存储层查不到数据则不写入缓存层。
	
# 缓存穿透将导致不存在的数据每次请求都要到存储层去查询，
	# 失去了缓存保护后端存储的意义。
	
# 造成缓存穿透的基本原因有两个:
	# 一: 自身业务代码或者数据出现问题。
	
	# 二: 一些恶意攻击、爬虫等造成大量空命中。
```

#### 缓存穿透问题解决方案

```java
// 1. 缓存空对象
String get (String key) {
  // 从缓存中获取数据
  String cacheValue = cache.get(key);
  // 缓存为空时
  if (StrUtils.isBlank(cacheValue)) {
    // 从存储中获取
    String storageValue = storage.get(key);
    cache.set(key, storageValue, 60 * 5);
    return storageValue;
  } else {
    return cacheValue;
  }
}
```

```shell
# 2. 布隆过滤器

# 对于恶意攻击，向服务器请求大量不存在的数据造成的缓存穿透，还可以用布隆过滤器先做一次过滤，
	# 对于不存在的数据，布隆过滤器一般都能过滤掉，不让请求再往后端发送。
	# 当布隆过滤器说 某个值存在 时，这个值可能不存在，
	# 当它说不存在时，那就肯定不存在。
```

![UTOOLS1589956558882.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1589956558882.png)

```shell
# 布隆过滤器就是一个 大型的位数组和几个不一样的无偏 hash 函数。
	
# 所谓 无偏 就是能够把元素的 hash 值算的比较均匀。

# 向布隆过滤器中添加 key 时，会使用多个 hash 函数对 key 进行 hash，
	# 算得一个 整数索引值 然后对 位数组长度 进行取模运算得到一个位置，
	# 每个 hash 函数都会算得一个不同的位置。
	# 再把 位数组 的这几个位置都置为 1，就完成了 add 操作。
	
# 向布隆过滤器询问 key 是否存在时，跟 add 一样，
	# 也会把 hash 的几个位置都算出来，看看 位数组 中这几个位置是否都为 1，
	# 只要要有一个位为 0，说明布隆过滤器中这个 key 不存在。
	# 如果都是 1，这并不能说明这个 key 就一定存在，只是极有可能存在，
	# 因为这些位置被置为 1 可能是因为其他的 key 存在所致。
	# 如果这个 位数组 比较稀疏，这个概率就会很大，相反同理。
	
# 这种方法适用于数据命中不高、数据相对固定、实时性低（通常是数据集较大）的应用场景
	# 代码维护较为复杂，但是缓存空间占用很少。
```

```shell
# 可以用 guvua 包自带的布隆过滤器，引入依赖:
```

```xml
<dependency>
	<groupId>com.google.guava</groupId> 
  <artifactId>guava</artifactId>
	<version>22.0</version>
</dependency>
```

```java
// 示例代码
import com.google.common.hash.BloomFilter;

// 初始化布隆过滤器
// 1000: 期望存入的数据个数
// 0.001: 期望的误差率
BloomFilter<String> bloomFilter = 
  BloomFilter.create(Funnels.stringFunnel(Charset.forName("utf-8")),
                    1000,
                    0.001);

// 把所有数据存入布隆过滤器
void init () {
  for (String key : keys) {
    bloomFilter.put(key);
  }
}

String get(String key) {
  // 从布隆过滤器这一级缓存判断下 key 是否存在
  Boolean exist = bloomFilter.mightContain(key);
  if (!exist) {
    return "";
  }
  
  // 从缓存中获取数据
  String cacheValue = cache.get(key);
  
  // 缓存为空
  if (StrUtils.isBlank(cacheValue)) {
    // 从存储中获取
    String storageValue = storage.get(key);
    cache.set(key, storageValue, 60 * 5);
    return storageValue;
  }
  return cacheValue;
}
```

#### 缓存失效

```shell
# 
```

