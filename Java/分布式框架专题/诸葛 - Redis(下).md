[TOC]

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

##### 集群是否支持批处理命令

```shell
# 3.0 不支持，即使在某些客户端下返回了值，很可能仅仅只是某一个节点的值

# 4.0 仅支持相同slot，key不能保证在相同slot还是没用
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
# 由于大批量缓存在同一时间失效，可能导致大量请求同时穿透缓存直达数据库，
	# 从而造成数据库瞬间压力过大甚至挂掉。
```

#### 缓存失效解决方案

```shell
# 在批量增加缓存时，最好将这一批数据的缓存过期时间设置为一个时间段内的不同时间。
```

```java
// 示例伪代码
String get(String key) {
  // 从缓存中获取数据
  String cacheValue = cache.get(key);
  
  // 缓存为空
  if (StrUtils.isBlank(cacheValue)) {
    // 从存储中获取
    String storageValue = storage.get(key);
    cache.set(key, storageValue, new Random().netInt(300) + 300);
    return storageValue;
  }
  
  return cacheValue;
}
```

#### 缓存雪崩

```shell
# 指缓存层支撑不住或宕机后，流量直接打向后端存储层。
	
# 由于缓存层承载着大量请求，有效保护存储层，但是如果缓存层由于某些原因不能提供服务，
	# 比如超大并发过来，缓存层支撑不住，
	# 或者由于缓存设计不好，类似大量请求访问 bigkey，导致缓存支撑的并发急剧下降。
	
# 于是大量流量直接到达存储层，存储层的调用量会暴增，造成存储层也会级联宕机的情况。
```

#### 缓存雪崩解决方案

```shell
# 预防和解决缓存雪崩问题，可以从以下三个方面进行着手。

# 1. 保证缓存层服务高可用性，比如使用 redis sentinel 或 redis cluster

# 2. 依赖隔离组件为后端限流并降级，比如使用 Hystrix。

# 3. 提前演练，在项目上线前，演练缓存层宕机后，应用以及后端的负载情况以及可能出现的问题，做以预案。
```

#### 热点缓存 key 重建优化

```shell
# 开发人员使用 "缓存 + 过期时间" 的策略，
	# 既可以加速数据读写，又保证数据的定期更新，这种模式基本能满足绝大部分需求。
	
# 但是有两个问题同时出现，可能就会对应用造成致命的危害。
	# 1. 当前 key 是一个热点 key，并发量非常大。
	# 2. 重建缓存不能在短时间完成，可能是一个复杂计算，例如复杂的 sql、多次 io、多个依赖等...
	
# 在缓存失效的瞬间，有大量线程来重建缓存，造成后端负载加大，甚至可能会让应用崩溃。

# 要解决这个问题，主要就是要避免大量线程同时重建缓存。
	# 可以利用互斥锁来解决，此方法只允许一个线程重建缓存，
	# 其他线程等待重建缓存的线程执行完，重新从缓存获取数据即可。
```

```java
// 示例伪代码:
String get (String key) {
  // 从 Redis 中获取数据
  String value = redis.get(key);
  
  if (value == null) {
    String mutexKey = "mutext:key:" + key;
    if (redis.set(mutexKey, "1", "ex 180", "nx")) {
      // 从数据源获取数据
      value = db.get(key);
      
      // 回写 Redis，并设置过期时间
      redis.setex(key, timeout, value);
      
      // 删除 key_mutex
      redis.delete(mutexkey);
    } else {
      // 其他线程休息 50 毫秒后重试
      Thread.sleep(50);
      get(key);
    }
  }
  return value;
}
```

## 开发规范与性能优化

### 键值设计

#### key 名设计

```shell
# 1.可读性和可管理型
	# 以业务名（或数据库名）为前缀（防止 key 冲突），用冒号分隔，
	# 比如 业务名:表名:id
trade:order:1

# 2.简洁性
	# 保证语义的前提下，控制 key 的长度，当 key 较多时，内存占用也不容忽视。
	# 例如:
user:{uid}:frinds:messages:{mid} 
	# 可以简化为:
u:{uid}:fr:m:{mid}

# 3.不要包含特殊字符
	# 例如: 空格、换行、单双引号、其他转义字符...
```

#### value 设计

##### 拒绝 bigkey

```shell
# 1.拒绝 bigkey（防止网卡流量、慢查询）
	# redis 中，一个字符串最大 512mb，
	# 一个二级数据结构（例如 hash、list、set、zset）可以存储 40亿个元素，
	# 但实际业务中，如果出现以下两种情况，就会认为是 bigkey
		# 1.字符串类型: 单个 value 值超过 10kb 就会被认为是 bigkey
		# 2.非字符串类型: 单个 key 包含元素超过 5000 个就会被认为是 bigkey
		
	# 非字符串的 bigkey，不要使用 del 删除，使用 hscan、sscan、zscan 方式渐进式删除，
	# 同时要注意防止 bigkey 过期时间自动删除问题，
	# 例如一个 200w 元素的 zset 自动过期时触发 del 操作，造成 redis 阻塞。
```

##### bigkey 的危害

```shell
# 1.导致 redis 阻塞

# 2.网络堵塞
	# bigkey 也就意味着每次获取要产生的网络流量较大，假设一个 bigkey 为 1mb，
	# 客户端每秒访问量为 1000，那么每秒产生 1000mb 的流量，
	# 对于普通的千兆网卡（按照字节算是128mb/s）的服务器来说简直是灭顶之灾，
	# 而且一般服务器会采用单机多实例的方式来部署，也就是说，
	# 一个 bigkey 可能会对其他实例也造成影响，后果不堪设想。

# 3.过期删除
	# bigkey 如果设置了过期时间，在过期时就会被删除，
	# 如果没有使用 redis4.0 的过期异步删除（lazyfree-lazy-expire yes）,
	# 就会存在阻塞 redis 的可能性。
```

##### bigkey 的产生

```shell
# 一般来说，bigkey 的产生都是由于程序设计不当，或者对于数据规模预料不清楚造成的

# 举例:
	# 社交类: 粉丝列表，如果某些明星或者大v 不精心设计下，必是bigkey。
	
	# 统计类: 按天存储某项功能或者网站的用户集合，除非没几个人用，否则必是 bigkey。
	
	# 缓存类: 将数据从数据库 load 出来序列化放到 redis里，可能乱七八糟一堆关联数据全缓存，就是 bigkey
```

##### 如何优化 bigkey

```shell
# 1.拆
	# big list: list1、list2、...listN
	
	# big hash: 可以将数据分段存储，比如一个大的 key，
		# 假设存了 100W 的用户数据，可以拆分成 200 个key，每个 key 下面存放 5k 个用户数据
		
# 2.如果 bigkey 不可避免，也要思考一下要不要每次把所有元素都取出来
	# 例如有时候仅仅需要 hmget，而不是 hgetall
	# 删除也是一样，尽量使用优雅的方式来处理。
	
# 3.选择适合的数据类型
	# 反例: set user:1:name tom | set user:1:age 19
	# 正例: hmset user:1 name tom age 19
	
# 4.控制 key 的生命周期
	# 数据进入缓存时，随机设置过期时间。
```

### 命令使用

#### O(N)命令关注N的数量

```shell
# 例: hgetall、lrange、smembers、zrange、sinter 等并非不能使用，
	# 但是需要明确 N 的值，
	# 有遍历的需求可以使用 hscan、sscan、zscan 代替。
```

#### 禁用命令

```shell
# 禁止线上使用 keys、flushall、flushdb 等高危命令。
```

#### 合理使用 select

```shell
# redis 的多数据库较弱，使用数字进行区分，很多客户端支持较差，
	# 同时多业务用多数据库实际还是单线程处理，会有干扰。
```

#### 使用批量操作提高效率

```shell
# 原生命令: 例如 mget、mset

# 非原生命令: 可以使用 pipeline 提高效率

# 但是要注意控制一次批量操作的元素的个数（例如 500 以内，实际也和元素字节数有关）

# 原生命令 和 非原生命令的不同:
	# 1.原生是原子操作，pipeline 是非原子操作。
	# 2.pipeline 可以打包不同的命令，原生做不到
	# 3.pipeline 需要客户端和服务端同时支持。
```

#### Redis 事务功能较弱，建议用 lua 替代

### 客户端使用

#### 避免多个应用使用一个 Redis 实例

```shell
# 正例: 不相干的业务拆分，公共数据做服务化。
```

#### 使用连接池

```shell
# 使用带有连接池的数据库，有效控制连接，同时提高效率，标准使用方式。
```

##### 连接数参数含义

| 参数名             | 含义                                                         | 默认值           | 使用建议                                                    |
| ------------------ | ------------------------------------------------------------ | ---------------- | ----------------------------------------------------------- |
| maxTotal           | 资源池中最大连接数                                           | 8                | 设置建议见下面                                              |
| maxIdle            | 资源池允许最大空闲的连接数                                   | 8                | 设置建议见下面                                              |
| minIdle            | 资源池确保最少空闲的连接数                                   | 0                | 设置建议见下面                                              |
| blockWhenExhausted | 当资源池用尽后，调用者是否要等待<br />只有为true 时，下面的 maxWaitMillis 才会生效 | true             | 建议使用默认值                                              |
| maxWaitMillis      | 当资源池连接用尽后，调用者的最大等待时间（单位毫秒）         | -1: 表示永不超时 | 不建议使用默认值                                            |
| testOnBorrow       | 向资源池借用连接时，<br />是否做连接有效性检测（ping），<br />无效连接会被移除 | false            | 业务量很大时，建议设置为false<br />省去每次多的 ping 的开销 |
| testOnReturn       | 向资源池归还连接时，<br />是否做连接有效性检测（ping），<br />无效连接会被移除 | alse             | 业务量很大时，建议设置为false<br />省去每次多的 ping 的开销 |
| jmxEnabled         | 是否开启jmx监控，可用于监控                                  | true             | 建议开启，但应用本身也要开启                                |

##### 优化建议

```shell
# maxTotal:
	# 考虑因素比较多，比如 业务希望 redis 并发量、客户端执行命令时间、
	# redis 资源（例如 nodes(redis 节点个数) * maxTotal 是不能超过 redis最大连接数 maxClients）
	# 资源开销，例如虽然希望控制空闲连接，但是不希望因为连接池的频繁释放创建连接造成不必要开销。
	
# 以一个例子说明，假设:
	# 一次命令时间（borrow|return resource + jedis 执行命令(含网络延迟)）的平均耗时约为 1ms,
	# 那么一个连接的 qps 大约是 1000，
	# 业务期望的 qps 是 50000，
	# 那么理论上就需要的资源池大小是 50000 / 1000 = 50个。
	# 但实际上，这是个理论值，还要考虑到要比理论值预留一些资源，通常 maxTotal 会比理论值更大。
	
# 但这个值，不是越大越好，一方面连接太多占用客户端和服务端资源，
	# 另一方面对于 redis 这种高 qps 的服务器，一个大命令的阻塞，
	# 即使设置再大的资源池仍然会无济于事。
```

```shell
# maxIdle 和 minIdle:

# maxIdle 实际上才是业务需要的最大连接数，maxTotal 是为了给出余量，
	# 所以 maxIdle 不要设置过小，否则会有很多创建连接的开销。
	
# 连接池的最佳性能是: maxTotal = maxIdle
	# 这样就避免连接池伸缩带来的性能干扰，
	# 但是如果并发量不大或 maxTotal 设置过高，会导致不必要的链接资源浪费。
	# 一般推荐 maxIdle 按上面的业务期望 qps 计算出来的理论连接数，maxTotal 可以再放大一倍。
	
# minIdle: 
	# 至少需要保持的空闲连接数，
	# 在使用连接的过程中，如果连接数超过了 minIdle，则继续建立连接，
	# 如果超过了 maxIdle，超过的连接执行完业务后会慢慢被移除连接池释放掉。
	
# 如果系统启动完，马上就会有很多的请求进来，那么可以给 redis 连接池做预热，
	# 比如快速的创建一些 redis 连接，执行简单命令，类似 ping()，
	# 快速的将连接池里的空闲连接提升到 minIdle 的数量。
```

##### 连接池预热示例代码

```java
public class preheatingTest {
  
  public static void main(String[] args) {
    List<Jedis> minIdleJedisList = new ArrayList<Jedis>(jedisPoolConfig.getMinIdle());
    // 预热
    for (int i = 0; i < jedisPoolConfig.getMinIdle(); i++) {
      Jedis jedis = null;
      try {
        jedis = pool.getResource();
        minIdleJedisList.add(jedis);
        jedis.ping();
      } catch (Exception e) {
        log.error("redis 连接获取异常")
      }
    }
    
    // 释放
    for (int i = 0; i < jedisPoolConfig.getMinIdle(); i++) {
      Jedis jedis = null;
      try {
        jedis = minIdleJedisList.get(i);
        jedis.close();
      } catch (Exception e) {
        log.error("redis 连接释放异常")
      }
    }
    
  }
  
}
```

```shell
# 总之，连接池的设置要根据实际系统的 qps 和调用 redis 客户端的规模整体评估每个节点所使用的资源来。
```

```shell
# 高并发下建议客户端添加熔断功能（例如 netflix hystrix）

# 设置合理的密码，如有必要可以使用 ssl 加密访问
```

```shell
# redis 对于过期键有三种清除策略:
	# 被动删除: 
		# 当 读/写 一个已经过期的 key 时，会触发惰性删除策略，直接删除掉这个过期 key。
		
	# 主动删除:
		# 由于 惰性删除策略 无法保证 冷数据 被及时删掉，
		# 所以 redis 会定期主动淘汰一批已过期的 key。
		
	# 当内存使用超过 maxMemory 限定时，触发主动清理策略。
		# 即 maxMemory-policy（最大内存淘汰策略）
		# 如果 redis 不设置 maxMemory，默认就是物理内存限制。
		# 当 redis 内存使用超出物理内存限制时，频繁与磁盘产生 swap 交换，性能急剧下降。
		
		# 默认策略是: volatile-lru
			# 即，使用 lru 算法删除 key，保证不过期数据不被删除，但是可能出现 OOM
			
		# allkeys-lru:
			# 根据 lru 算法删除 key,不管数据有无超时，直到腾出足够空间为止
			
		# allkeys-random:
			# 随机删除所有键，直到腾出足够空间为止
			
		# volatile-random:
			# 随机删除过期键，知道腾出足够空间为止
			
		# volatile-ttl:
			# 根据键值对象的 ttl 属性，删除最近将要过期数据，如果没有，回退到 noeviction 策略。
			
		# noeviction:
			# 不删除任何数据，拒绝所有写入操作，并返回 OOM 异常，redis 只响应读操作。
		
# 注意:
	# 当 redis 运行在主从模式时，只要主节点才会执行被动和主动这两种过期删除策略，
	# 然后把删除操作 del key 同步到从节点。
```

## Redis 分布式锁

### 简单项目搭建

```xml
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>2.1.2.RELEASE</version>
  <relativePath /> <!-- lookup parent from repository -->
</parent>

<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  <java.version>1.8</java.version>
</properties>

<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
  </dependency>

  <dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.6.5</version>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
    </plugin>
  </plugins>
</build>
```

```yaml
server:
  port: 8090

spring:
  redis:
    host: 127.0.0.1
    port: 6379
```

### 为什么需要分布式锁代码演示

```shell
# 在 redis 预设 stock 50 的 kv
```

```java
@RestController
public class IndexController {

    private static final Logger log = LoggerFactory.getLogger(IndexController.class);

    @Autowired
    private StringRedisTemplate stringRedisTemplate;

    @GetMapping("/deductStock")
    public String deductStock() {
        String value = stringRedisTemplate.opsForValue().get("stock");
        int stock = Integer.parseInt(value);

        if (stock > 0) {
            // 自减
            stringRedisTemplate.opsForValue().set("stock", --stock + "");
            log.info("扣减成功，剩余库存: {}", stock);
        } else {
            log.error("扣减失败，库存不足");
        }

        return "success";
    }

}
```

```shell
# 上述代码存在的问题:
	# 假设现在 N 个线程同时进入方法，拿到 stock 值为 50 保存在线程本地内存中，
	# 1 线程先执行自减，设值到 redis 中，此时为 49，
	# 2 线程再执行自减，设值到 redis 中，此时仍然为 49.
	# 存在类似 线程间通讯 的问题，如果是 Java 层面的变量，可以用 voliate + atomic 关键字解决。
	# 这里可以用 synchronized(this) 解决单机架构，但是效率比较低。
	# 如果是分布式架构，存在多个服务实例并行处理该程序请求，Java 层面的锁就无法解决问题了。
```

```shell
# 问题演示:
	# 启动两个服务实例（切换不同端口）
	# 配置 Nginx 负载，Jmeter 压测，即可出现问题。
```

![UTOOLS1590117183465.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1590117183465.png)

![UTOOLS1590117212334.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1590117212334.png)

### Redis setnx

```java
@GetMapping("/deductStock/setNx")
    public String deductStockSetNx() {

        String lockKey = "lockKey";

        try {
            /**
             * 这里存在一个问题:
             *  如果这里的业务执行时间超出了 redis key 失效时间
             *  A -> 执行到一半 -> 锁被redis 清除了
             *  B -> 进来了，加锁 -> 执行业务
             *  A -> 执行完了，删除锁 -> B 的锁又没了
             *  C -> 进来...
             */
            Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "lock", 3, TimeUnit.SECONDS);
            if (result) {
                String value = stringRedisTemplate.opsForValue().get("stock");
                int stock = Integer.parseInt(value);

                if (stock > 0) {
                    // 自减
                    stringRedisTemplate.opsForValue().set("stock", --stock + "");
                    log.info("扣减成功，剩余库存: {}", stock);
                } else {
                    log.error("扣减失败，库存不足");
                }
                return "success";
            } else {
                log.error("没有获取到锁，线程抛弃");
                return "error";
            }
        } finally {
            stringRedisTemplate.delete(lockKey);
        }

    }
```

```java
@GetMapping("/deductStock/setNx")
    public String deductStockSetNx() {

        String lockKey = "lockKey";
        
        String clientId = UUID.randomUUID().toString();

        try {
            Boolean result = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, clientId, 3, TimeUnit.SECONDS);
            if (result) {
                String value = stringRedisTemplate.opsForValue().get("stock");
                int stock = Integer.parseInt(value);

                if (stock > 0) {
                    // 自减
                    stringRedisTemplate.opsForValue().set("stock", --stock + "");
                    log.info("扣减成功，剩余库存: {}", stock);
                } else {
                    log.error("扣减失败，库存不足");
                }
                return "success";
            } else {
                log.error("没有获取到锁，线程抛弃");
                return "error";
            }
        } finally {
            // 这里解决了每个线程锁防止被其他线程删除的问题
            // 但是还会产生线程还没执行完，锁就被自动清理了的问题
            if (clientId.equals(stringRedisTemplate.opsForValue().get(lockKey))) {
                stringRedisTemplate.delete(lockKey);
            }
        }

    }
```

### Redisson

```java
 		@Bean
    public Redisson redisson() {
        // 单机模式
        Config config = new Config();
        config.useSingleServer().setAddress("redis://127.0.0.1:6379").setDatabase((0));
        return (Redisson) Redisson.create(config);

//        // 集群模式
//        config.useClusterServers()
//                .addNodeAddress("redis://ip:port")
//                .addNodeAddress("redis://ip:port")
//                ...;
    }
```

#### Redisson 分布式锁实现原理

![UTOOLS1590129569423.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1590129569423.png)

#### Redis Lua脚本

```shell
# Redis 在 2.6 版本推出了脚本功能，允许开发者使用 Lua 语言编写脚本传到 Redis 中执行。

# 使用 Lua 脚本的好处如下:
	# 减少网络开销:
		# 本来 5 次网络请求的操作，可以用一个请求完成，
		# 原先 5 次请求的逻辑放在 redis 服务器上完成。
		# 使用脚本，减少了网络往返时延，这点跟管道类似。
		
	# 原子操作:
		# Redis 会将整个脚本作为一个整体执行，中间不会被其他命令插入。
		# 管道不是原子的，不过 redis 的批量操作命令是原子的，类似 mset。
		
	# 替代 redis 的事务功能:
		# redis 自带的事务功能很鸡肋，报错不支持回滚。
		# 而 lua 脚本几乎实现了常规的事务功能，支持报错回滚操作。
		# 官方推荐使用 lua 脚本替代 redis 的事务功能。
```

```shell
# Redis2.6 版本后，通过内置的 Lua 解释器，可以使用 EVAL 命令对 Lua 脚本进行求值。

# EVAL 命令的格式如下:
EVAL script numkeys key [key ...] arg [arg ...]

# script 参数是一段 Lua 脚本程序，它会被运行在 Redis 服务器上下文中，
	# 这段脚本 不必(也不应该) 定义为一个 Lua 函数。
	
# numkeys 参数用于指定键名参数的个数。
	# 键名参数 key [key ...] 从 EVAL 的第三个参数开始算起，
	# 表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 KEYS 数组，
	# 用 1 为基址的形式访问 KEYS[1]、KEYS[2]、... 依次类推。
	
# 在命令的最后，那些不是键名参数的附加参数 arg [arg...]，
	# 可以在 Lua 中通过全局变量 ARGV 数组访问，
	# 访问的形式和 KEYS 变量类似 ARGV[1]、ARGV[2]、... 诸如此类。
```

![UTOOLS1590132611310.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/UTOOLS1590132611310.png)

```shell
# 其中 "return {KEYS[1], KEYS[2], ARGV[1], ARGV[2]}" 是被求值的 Lua 脚本。
	
# 数字 2 指定了键名参数的数量，
	# key1 和 key2 是键名参数，分别使用 KEYS[1] 和 KEYS[2] 访问。
	
# 而最后的 first 和 second 则是附加参数，
	# 可以通过 ARGV[1] 和 ARGV[2] 访问它们。
	
# 在 Lua 脚本中，可以使用 redis.call() 函数来执行 Redis 命令。
```

```java
public void test() {
  // 商品库存 key
  String key = "product_stock_1";
  // 初始化商品的库存
  jedis.set(key, "15");
  
 /**
  * 模拟一个商品减库存的 Lua 脚本原子操作
  * lua 脚本命令执行方式 redis-cli --eval /tmp/test.lua , 10
  */
  String script = " local count = redis.call('get', KEYS[1]) " +
    " local a = tonumber(count) " + 
    " local b = tonumber(ARGV[1]) " + 
    " if a >= b then " +
    " redis.call('set', KEYS[1], count-b) " +
    " return 1 " + 
    // 这里注释解开就可以模拟异常，事务回滚
    // 执行报错，Lua 中变量赋值是用 = 而不是 ==
    // " bb == 0 " +
    " end " +
    " return 0 ";
  
  Object obj = jedis.eval(script, Arrays.asList(key), Arrays.asList("10"));
  System.out.println(obj);
}
```

```shell
# 注意:
	# 不要在 Lua 脚本中出现 死循环 和 耗时的运算，否则 redis 会阻塞，将不接受其他命令。
	
	# redis 是单进程、单线程执行脚本，管道不会阻塞 redis。
```

#### Redisson 实现分布式锁源码

```java
		@Override
    public void lockInterruptibly() throws InterruptedException {
        lockInterruptibly(-1, null);
    }

    @Override
    public void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException {
        long threadId = Thread.currentThread().getId();
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // lock acquired
        if (ttl == null) {
            return;
        }

        RFuture<RedissonLockEntry> future = subscribe(threadId);
        commandExecutor.syncSubscription(future);

        try {
            while (true) {
                ttl = tryAcquire(leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    break;
                }

                // waiting for message
                if (ttl >= 0) {
                    getEntry(threadId).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    getEntry(threadId).getLatch().acquire();
                }
            }
        } finally {
            unsubscribe(future, threadId);
        }
//        get(lockAsync(leaseTime, unit));
    }

		private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(leaseTime, unit, threadId));
    }
    
    private RFuture<Boolean> tryAcquireOnceAsync(long leaseTime, TimeUnit unit, final long threadId) {
        if (leaseTime != -1) {
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        }
        RFuture<Boolean> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        ttlRemainingFuture.addListener(new FutureListener<Boolean>() {
            @Override
            public void operationComplete(Future<Boolean> future) throws Exception {
                if (!future.isSuccess()) {
                    return;
                }

                Boolean ttlRemaining = future.getNow();
                // lock acquired
                if (ttlRemaining) {
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;
    }

    private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, final long threadId) {
        if (leaseTime != -1) {
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        ttlRemainingFuture.addListener(new FutureListener<Long>() {
            @Override
            public void operationComplete(Future<Long> future) throws Exception {
                if (!future.isSuccess()) {
                    return;
                }

                Long ttlRemaining = future.getNow();
                // lock acquired
                if (ttlRemaining == null) {
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;
    }

		<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        internalLockLeaseTime = unit.toMillis(leaseTime);

        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                    Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```

```shell
# Redission 将资源名作为 Hash 的 Key，将当前线程 ID 作为 Field，值为 1
	# 用这样的 Hash 结构存储锁。
	
# Redisson 在设置锁成功后，会进入续命锁的逻辑，在超时时间 / 3 （默认超时时间是30秒，所以在10秒就开始续命）
	#	成功后添加监听，继续轮询调用锁续命。
	
# Redisson 是支持可重入锁的，在同 Key 同 Field 的情况下，执行 incrby 的操作

# Redisson 其他线程没有拿到锁时，会拿到锁的过期时间，然后进入自旋尝试获取锁。
	
# Redisson 内所有 redis 操作都是 Lua 脚本原子性的
```

```shell
# Redis 在多实例架构情况时，可能存在主从同步还未完成，主节点就宕机了，
	# 此时其他线程又通过新选举的主节点拿到锁，造成 bug 的问题。
	
# 如果用 zookeeper 是不会出现这样的问题的，因为 zk 是强一致性的设计，必须所有节点同步完成才会返回成功。

# Redssion 也提供了 RedLock 的实现 Api(就是往所有 redis 节点发命令，超过半数成功返回才算成功)
	# 不建议使用。
```

#### Redis 高并发分布式锁

```shell
# 分段锁。
```

