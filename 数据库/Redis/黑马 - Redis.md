# 黑马 - Redis

## Redis 入门

### NoSQL

```shell
# NoSQL: 即 Not-Only SQL（泛指非关系型的数据库）
	# 作用: 应对基于海量用户和海量数据前提下的数据处理问题。
	
	# 特征:
		# 可扩容、可伸缩
		# 大数据量下高性能
		# 灵活的数据模型
		# 高可用
	
# 常见 NoSQL 数据库:
	# Redis
	# Memcache
	# Hbase
	# MongoDB
```

### 应用场景

```shell
# 电商为例:
	# 商品基本信息（MySQL）
		
	# 商品附加信息（MongoDB）
	
	# 图片信息（分布式文件系统）
	
	# 搜索关键字（ElasticSearch、Solo）
	
	# 热点信息（Redis）
```

### Redis 简介

```shell
# 概念: 
	# Redis 是用 C 语言开发的一个开源的高性能键值对数据库
	
# 特征:
	# 数据间没有必然的关联关系

	# 内部采用单线程机制进行工作
	
	# 高性能，官方提供测试数据为:
		# 50 个并发执行 100000 个请求，读的速度是 110000 次/s，写的速度是 81000 次/s
		
	# 多数据类型支持
		# String
		# list
		# hash
		# set
		# sorted_set
		
	# 持久化支持，可以进行数据灾难恢复
```

### Redis 的应用

```shell
# 为热点数据加速查询

# 任务队列，秒杀、抢购、购票排队等

# 即时信息查询，如各位排行榜、各类网站访问统计、公交到站信息、在线人数信息、设备信号等

# 时效性信息控制、验证码控制、投票控制等

# 分布式数据共享，如分布式集群架构中的 Session 分离

# 消息队列

# 分布式锁
```

### Redis 的基本操作

```shell
# 安装略
```

#### 命令行工具

```shell
# 设置 key value 数据
set key value				# 举例: set name swordsman

# 根据 key 查询对应的 value，如果不存在，返回空（nil）
get key 						# 举例: get name

# 清除屏幕信息
clear

# 退出命令行
exit

# 获取命令帮助文档，获取组中所有命令信息名称
help 命令名称
help @组名
```

![UTOOLS1576462339176.png](https://img04.sogoucdn.com/app/a/100520146/5c091307d535a10a9205706ddf8c36d9)

## Redis 数据类型

### Redis 数据存储格式

```shell
# redis 自身是一个 Map，所有的数据都是采用 key:value 的形式存储

# 数据类型 指的是存储的数据的类型，也就是 value 部分的类型，key 部分永远都是字符串
```

### String

```shell
# 存储的数据: 单个数据，最简单的数据存储类型，也是最常用的数据存储类型

# 存储的数据格式: 一个存储空间保存一个数据

# 存储内容: 通常使用字符串，如果字符串以整数的形式展示，可以作为数字操作使用
```

#### String 类型数据的基本操作

```shell
# 添加/修改 数据
set key value

# 获取数据
get key

# 删除数据
del key

# 添加/修改多个数据
mset key1 value1 key2 value2 ...

# 获取多个数据
mget key1 key2

# 获取数据字符个数（字符串长度）
strlen key

# 追加信息到原始信息后部（如果原始信息存在就追加，否则新建）
append key value
```

![UTOOLS1576463143893.png](https://img02.sogoucdn.com/app/a/100520146/b3e523689e2842217032907c527d2a9e)

#### String 类型数据的扩展操作

```shell
# 设置数值数据增加指定范围的值
incr key	# 自增1
incrby key increment	# 自增指定数字
incrbyfloat key increment	# 自增指定浮点数

# 设置数值数据减少指定范围的值
decr key
decr key increment
```

```shell
# String 作为数值操作:
	# String 在 Redis 内部存储默认是字符串，当遇到 incr、decr 时会转成数值型进行计算
	
	# Redis 所有的操作都是原子性的，采用单线程处理所有业务，命令是一个一个执行的，无需考虑并发
	
	# 注意:
		# 按数值进行操作的数据，如果原始数据不能转成数值，或超越了 Redis 数值上限范围，将报错。
		# 9223372036854775807 (java中long型数据最大值，Long.MAX_VALUE)

# Tips 1:
	# Redis 用于控制数据库表主键 id，为数据库表主键提供生成策略，保障数据库表的主键唯一性
	# 此方案适用于所有数据库，且支持数据库集群
```

```shell
# 业务场景:
	# "最强女生" 启动海选投票，只能通过微信投票，每个微信号每 4 小时只能投一票
	
	# 电商商家开启热门商品推荐，热门商品不能一直处于热门期，每种商品热门期维持 3 天，3天后自动取消热门
	
	# 新闻网站会出现热点新闻，热点新闻最大的特征是时效性，如果自动控制热点新闻的时效性

# 解决方案:
	# 设置数据具有指定的生命周期
	setex key seconds value
	psetex key milliseconds value		# 以毫秒为单位设置 key 的生存时间。
	
# Tips 2:
	# Redis 控制数据的生命周期，通过数据是否失效控制业务行为，适用于所有具有时效性限定控制的操作
```

#### String 类型数据操作的注意事项

```shell
# 数据操作不成功的反馈与数据正常操作之间的差异
	# 表示运行结果是否成功:
		# (integer)0 -> false : 失败
		# (integer)1 -> true : 成功
		
	# 表示运行结果值:
		# (integer)3 -> 3 : 3个
		# (integer)1 -> 1 : 1个
		
	# 数据未获取到:
		# (nil) 等同于 null
		
	# 数据最大存储量:
		# 512 MB
```

#### String 类型应用场景

```shell
# 业务场景:
	# 主页高频访问信息显示控制，例如新浪微博 大V 主页显示粉丝数与微博数量
	
# 解决方案:
	# 在 Redis 中为大V 用户设定用户信息，以用户主键和属性值作为 key，后台设定定时刷新策略即可
    # eg: user:id:1:fans -> 100
    # eg: user:id:1:blogs -> 200
    # eg: user:id:1:focuss -> 300
    
   # 在Redis 中以 json 格式存储大V 用户信息，定时刷新（也可以使用 hash 类型)
   	# eg: user:id:1 -> {"id":1, "name":"agefades", "fans":100, "blogs": 200}
   	
# Tips 3:
	# Redis 应用于各种结构型和非结构型高热度数据访问加速
```

#### Key 的设置约定

```shell
# 数据库中的热点数据 key 命名惯例:
	# 表名:主键名:主键值:字段名
	# eg: order:id:1:name
```

### Hash

#### 存储的困惑

```shell
# 对象类数据的存储如果具有较频繁的更新需求操作会显得笨重
```

#### Hash 类型

```shell
# 新的存储需要: 对一系列存储的数据进行编组，方便管理，典型应用存储对象信息

# 需要的存储结构: 一个存储空间保存多个键值对数据

# Hash 类型: 底层使用哈希表结构实现数据存储
```

![UTOOLS1576467626630.png](https://img01.sogoucdn.com/app/a/100520146/8ab7e54bba4aa8d3919ebe7d7f9de585)

#### Hash 类型数据的基本操作

```shell
# 添加/修改数据
hset key field value

# 获取数据
hget key field
hgetall key

# 删除数据
hdel key field1 [field2]

# 添加/修改多个数据
hmset key field1 value1 field2 value2

# 获取多个数据
hmget key field1 field2

# 获取哈希表中字段的数量
hlen key

# 获取哈希表中是否存在指定的字段
hexists key field
```

#### Hash 类型数据扩展操作

```shell
# 获取哈希表中所有的字段名或字段值
hkeys key
kvals key

# 设置指定字段的数值数据增加指定范围的值
hincrby key field increment
hincrbyfloat key field increment
```

#### Hash 数据类型操作的注意事项

```shell
# hash 类型下的 value 只能存储字符串，不允许存储其他数据类型
	# 不存在嵌套现象，如果数据未获取到，对应的值为 nil
	
# 每个 hash 可以存储 2^32 - 1 个键值对

# hash 类型十分贴近对象的数据存储形式，并且可以灵活添加删除对象属性。
	# 但 Hash 设计初衷不是为了存储大量对象而设计的，切记不可滥用，更不可以将 hash 作为对象列表使用
	
# hgetall 操作可以获取全部属性，如果内部 field 过多，遍历整体数据效率就会很低
	# 有可能成为数据访问瓶颈
```

#### Hash 类型应用场景

![UTOOLS1576474134777.png](https://img01.sogoucdn.com/app/a/100520146/a53d00d79f0a889821bd9131d10cc6fc)

```shell
# 解决方案:
	# 以客户id 作为 key，每位客户创建一个 hash 存储结构存储对应的购物车信息
	# 将商品编号作为 field，购买数量作为 value 进行存储
	# 添加商品，追加全新的 field 与 value
	# 浏览: 遍历 hash
	# 更改数量: 自增/自减，设置 value 值
	# 删除商品，field
	# 清空: 删除 Key

# 当前设计仅仅将数据存储到 redis，并没有起到加速的作用，用户二次进入购物车仍需查库
	# 解决方案:
		# 每条购物车中的商品记录保存两条 field
		# 商品id:nums  :  保存数值
		# 商品id:info  : 	保存商品信息
		
		# hsetnx key field value  :  仅当哈希字段不存在时，才设置该字段的值
```

```shell
# Redis Hash 应用于购物车数据存储设计

# Redis Hash 应用于抢购，限购类、限量发放优惠券、激活码等业务的数据存储设计
```

### List

```shell
# 数据存储需求: 存储多个数据，并对数据进入存储空间的顺序进行区分

# 需要的存储结构: 一个存储空间保存多个数据，且通过数据可以体现进入顺序

# list 类型: 保存多个数据，底层使用双向链表存储结构实现
```

#### List 类型数据基本操作

```shell
# 添加/修改数据
lpush key value1 [value2] ...
rpush key value1 [value2] ...

# 获取数据
lrange key start stop	 # 从列表中获取元素范围
lindex key index	# 通过元素的索引从列表中获取元素	
llen key	# 获取列表的长度

# 获取并移除数据
lpop key
rpop key
```

#### list 类型数据扩展操作

```shell
# 规定时间内获取并移除数据
blpop key1 [key2] timeout
brpop key1 [key2] timeout
```

#### 业务场景

```shell
# 微信朋友圈点赞，要求按照点赞顺序显示点赞好友信息，如果取消点赞，移除对应好友信息

# 解决方案:
	# 移除指定数据:
	lrem key count value	# 根据参数 COUNT 的值，移除列表中与参数 Value 相等的元素
	
# Tips 6:
	# Redis List 应用于具有操作先后顺序的数据控制
	

# 微博中个人用户的关注列表需要安装用户的关注顺序进行展示，粉丝列表需要将最近关注的粉丝列在前面

# 新闻、咨询类网站如何将最新的新闻或咨询按照发生的时间顺序展示？

# 企业运营过程中，系统将产生大量的运营数据，如何保障多台服务器操作日志的统一顺序输出?

# 解决方案:
	# 依赖 list 的数据具有顺序的特征对信息进行管理
	# 使用队列模型解决多路信息汇总合并的问题
	# 使用栈模型解决最新消息的问题
	
# Tips 7:
	# Redis 应用于最新消息展示
```

#### List 类型数据操作注意事项

```shell
# List 中保存的数据都是 String 类型的，数据总容量是有限的，最多 2^32 - 1 个元素

# List 具有索引的概念，但是操作数据时通常以队列的形式进行入队出队操作，或以栈的形式进行入栈出栈操作

# 获取全部数据操作结束索引设置为 -1

# list 可以对数据进行分页操作，通常第一页的信息来自于 list
	# 第二页及更多的信息通过数据库的形式加载
```

### Set

```shell
# 新的存储需求: 存储大量的数据，在查询方面提供更高的效率

# 需要的存储结构: 能够保存大量的数据，高效的内部存储机制，便于查询

# set 类型: 与 hash 存储结构完全相同，仅存储键，不存储值（nil）并且值不允许重复
```

#### Set 类型数据的基本操作

```shell
# 添加数据
sadd key member1 [member2]

# 获取全部数据
smembers key

# 删除数据
srem key member1 member2

# 获取集合数据总量
scard key

# 判断集合中是否包含指定数据
sismember key member
```

#### Set 类型数据的扩展操作

```shell
# 业务场景:
	# 每位用户首次使用今日头条时，会设置 3 项爱好的内容，但是后期为了增加用户的活跃度、兴趣点，必须让用户对其他信息类别逐渐产生兴趣，增加客户留存度，如何实现？
	
# 业务分析: 
	# 系统分析出各个分类的最新或最热点信息条目并组成 set 集合
	# 随机挑选其中部分信息
	# 配合用户关注信息分类中的热点信息组织成展示的全信息集合
	
# 解决方案:
	# 随机获取集合中指定数量的数据
	srandmember key [count]
	
	# 随机获取集合中的某个数据并将该数据移除集合
	spop key [count]
	
# Tips 8:
	# Redis Set 应用于随机推荐类信息检索，例如热点歌单推荐，热点新闻推荐..
```

```shell
# 业务场景:
	# 脉脉为了促进用户间的交流，保障业务成单率的提升，需要让每位用户拥有大量的好友...
	
	# 美团为了提升成单量，必须帮助用户挖掘美食需求，如果推荐给用户最适合自己的美食...
	
# 解决方案:
	# 求两个集合的交、并、差集
	sinter key1 [key2]
	sunion key1 [key2]
	sdiff key1 [key2]
	
	# 求两个集合的交、并、差集并存储
	sinterstore destination key1 [key2]
	suinonstore destination key1 [key2]
	sdiffstore destination key1 [key2]
	
	# 将指定数据从原始集合中移动到目标集合中
	smove source destination member
	

# Tips 9:
	# Redis 应用于同类信息的关联搜索，二度关联搜索，深度关联搜索
	# 显示共同关注（一度）
	# 显示共同好友（一度）
	# 由用户 A 出发，获取到好友用户 B 的好友信息列表（一度）
	# 由用户 A 出发，获取到好友用户 B 的购物清单列表（二度）
	# 由用户 A 出发，获取到好友用户 B 的游戏充值列表（二度）
```

```shell
# 业务场景:
	# 公司对旗下新的网站做推广，统计网站的 PV、UV、IP
		# PV: 网站被访问次数，可通过刷新页面提高访问量
		# UV: 网站被不同用户访问次数，可通过 cookie 统计访问量，相同用户切换 ip 地址，UV 不变
		# IP: 网站被不同 IP 地址访问的总次数,可通过IP 地址统计访问量，相同 IP 不同用户访问,IP 不变
		
# 解决方案:
	# 利用 set 集合的数据去重特征，记录各种访问数据
	# 建立 String 类型数据，利用 incr 统计日访问量（PV）
	# 建立 set 模型，记录不同 cookie 数量（UV）
	# 建立 set 模型，记录不同 IP 数量（）(IP)
	

# Tips 10:
	# Redis Set 应用于同类型数据的快速去重
```

```shell
# 业务场景:
	# 黑名单/白名单
	
# 解决方案:
	# 基于经营战略设定问题用户发现、鉴别规则
	# 周期性更新满足规则的用户黑名单，加入 set 集合
	# 用户行为信息达到后与黑名单进行对比，确认行为去向
	# 黑名单过滤 ip 地址: 应用于开放游客访问权限的信息源
	# 黑名单过滤设备信息: 应用于限定访问设备的信息源
	# 黑名单过滤用户: 应用于基于访问权限的信息源
	
	
# Tips 12:
	# Redis 应用于基于黑名单与白名单设定的服务控制
```

#### Set 类型数据操作的注意事项

```shell
# set 类型不允许数据重复，如果添加的数据在 set 中已经存在，将只保留一份

# set 虽然与 hash 的存储结构相同，但是无法启用 hash 中存储值的空间
```

## sorted_set

```shell
# 新的存储需求: 数据排序有利于数据的有效展示，需要提供一种可以根据自身特征进行排序的方式
# 需要的存储结构: 新的存储模型，可以保存可排序的数据
# Sorted_set 类型: 在 set 的存储结构基础上添加可排序字段
```

#### sorted_set 类型数据的基本操作

```shell
# 添加数据:
zadd key score1 member1 [score2 member2]

# 获取全部数据:
zrange key start stop [WITHSCORES]	# 按索引返回已排序集合中的成员范围，scores 从小到大
zrevrange key start stop [WITHSCORES] # scores 从大到小

# 删除数据:
zrem key member [member]...

# 按条件获取数据
zrangebyscore key min max [WITHSCORES] [LIMIT]
zrevrangebyscore key max min [WITHSCORES]

# 条件删除数据
zremrangebyrank key start stop
zremrangebyscore key min max

# 注意:
	# min 与 max 用于限定搜索查询的条件
	# start 与 stop 用于限定查询范围，作用于索引，表示开始和结束索引
	# offset 与 count 用于限定查询范围，作用于查询结果，表示将开始位置和数据总量
	
# 获取集合数据总量:
zcard key
zcount key min max

# 集合交、并操作
zinterstore destination numkeys key [key ...]
zunionstore destination numkeys key [key ...]
```

#### sorted_set 类型数据的扩展操作

```shell
# 业务场景:
	# 票选广东十大杰出青年，各类综艺选秀海选投票
	# 各类资源网站 Top 10
	# 聊天室活跃度统计
	# 游戏好友亲密度
	
# 业务分析:
	# 为所有参与排名的资源建立排序依据
	
# 解决方案:
	# 获取数据对应的索引（排名）
	zrank key member
	zrevrank key member
	
	# score 值获取与修改
	zscore key member
	zincrby key increment member
	
# Tips 13:
	# Redis sorted_set 应用于计数器组合排序功能对应的排名
```

```shell
# 业务场景:
	# 云盘下载体验 vip 到期后，如何有效管理此类信息？
	# 网站定期开启投票、讨论、限时进行，逾期作废，如何有效管理此类过期信息？
	
# 解决方案:
	# 对于基于时间限定的任务处理，将处理时间记录为 score 值，利用排序功能区分处理的先后顺序
	# 记录下一个要处理的时间，当到期后处理对应任务，移除 Redis 中的记录，并记录下一个要处理的时间
	# 当新任务加入时，判定并更新当前下一个要处理的任务时间
	# 为提升 sorted_set 的性能，通常将任务根据特征存储成若干个 sorted_set，
		# 例如 1小时内，1天内，周内...
		# 操作时逐级提升，将即将操作的若干个任务纳入到 1小时内处理的队列中
		
	# 获取当前系统时间:	time
	
# Tips 14:
	# Redis sorted_set 应用于定时任务执行顺序管理或任务过期管理
```

```shell
# 业务场景:
	# 任务/消息权重设定应用
		# 当任务或者消息待处理，形成了任务队列或消息队列时
			# 对于高优先级的任务要保障对其优先处理，如何实现任务权重管理
			
# 解决方案:
	# 对于带有权重的任务，优先处理权重高的任务，采用 score 记录权重即可
	# 多条件任务权重设定:
		# 如果权重条件过多时，需要对排序 score 值进行处理，保障 score 值能够兼容 2 条件或多条件
			# 例如外贸订单优先于国内订单，总裁订单优先于员工订单...
			
		# 因 score 长度受限，需要对数据进行拦断处理，尤其是时间设置为小时或分钟级即可
		
		# 先设定订单类别，后设定订单发起角色类别，整体 score 长度必须是统一的，不足位补0
			# 第一排序规则首位不得是 0
				# 例如 外贸101，国内102，经理004，员工008
				# 员工下的外贸单 score 值为 101008（优先）
				# 经理下的国内单 score 值为 102004
				
# Tips 15:
	# Redis sorted_set 应用于即时任务/消息队列执行管理
```

#### sorted_set 类型数据操作的注意事项

```shell
# score 保存的数据存储空间是 64 位，如果是整数范围是-9007199254740992~9007199254740992

# score 保存的数据也可以是一个双精度的 double 值，基于双精度浮点数的特征，可能会丢失精度，使用时要慎重

# sorted_set 底层存储还是基于 set 结构的，因此数据不能重复
	# 如果重复添加相同数据，score 值将被反复覆盖，保留最后一次修改的结果
```

### Redis 解决方案列表

![UTOOLS1576488095721.png](https://img03.sogoucdn.com/app/a/100520146/3295814ec1f28aaf7da1b6f3d5d7b189)

## Redis 通用操作

### Key 通用指令

#### Key 特征

```shell
# Key 是一个字符串，通过 Key 获取 Redis 中保存的数据
```

#### Key 应该设计哪些操作

```shell
# 对于 key 自身状态的相关操作，例如: 删除、判定存在、获取类型等...

# 对于 key 有效性控制相关操作，例如: 有效期设定，判定是否有效，有效状态的切换等

# 对于 key 快速查询操作，例如: 按指定策略查询 key
```

#### Key 基本操作

```shell
# 删除指定 key
del key

# 获取 key 是否存在
exists key

# 获取 key 的类型
type key
```

#### Key 扩展操作（时效性控制）

```shell
# 为指定 key 设置有效期
expire key secondes
pexpire key milliseconds
expireat key timestamp
pexpireat key milliseconds-timestamp

# 获取 key 的有效时间
ttl key
pttl key

# 切换 key 从时效性转换为永久性
persist key
```

#### Key 扩展操作（查询模式）

```shell
# 查询 key
keys pattern

# 查询模式规则:
```

![UTOOLS1576547348995.png](https://img01.sogoucdn.com/app/a/100520146/2fcbed791ffcf73ef006c4642574132a)

#### Key 其他操作

```shell
# 为 Key 改名
rename key newkey
renamenx key newkey

# 对所有 Key 排序
sort

# 其他 Key 通用操作
help @generic
```

### 数据库通用操作

#### Key 的重复问题

```shell
# Key 是由程序员定义的，在使用过程中，会出现大量数据以及对应 Key
	# 数据不区分种类、类别混杂在一起，极易出现重复或冲突
	
# 解决方案
	# Redis 为每个服务提供有 16 个数据库，编号从 0 - 15
	# 每个数据库之间的数据相互独立
```

#### db 基本操作

```shell
# 切换数据库
select index

# 其他操作
quit
ping
echo message

# 数据移动
move key db

# 数据清除
dbsize
flushdb
flushall
```

## Redis 持久化

### 持久化简介

#### 什么是持久化

```shell
# 利用永久性存储介质将数据进行保存，在特定的时间将保存的数据进行恢复的工作机制称为永久化。
```

#### 为什么要进行持久化

```shell
# 防止数据的意外丢失，确保数据安全性
```

#### 持久化过程保存什么

```shell
# 将当前数据状态进行保存，快照形式，存储数据结果，存储格式简单，关注点在数据

# 将数据的操作过程进行保存，日志形式，存储操作过程，存储格式复杂，关注点在数据的操作过程
```

![UTOOLS1576548037013.png](https://img01.sogoucdn.com/app/a/100520146/c9aa75ab8bfecf55cb37d89b5134991c)

### RDB

#### save 指令

```shell
# 命令
save

# 作用
	# 手动执行一次保存操作

## save 指令相关配置

# 设置本地数据库文件名，默认值为 dump.rdb，通常设置为 dump-端口号.rdb
dbfilename dump.rdb	

# 存储 .rdb 文件的路径，通常设置成存储空间较大的目录，目录名称为 data
dir

# 设置存储至本地数据库时是否压缩数据，默认为 yes，采用 LZF 压缩
# 通常默认为开启状态，如果设置为 no，可以节省 CPU 运行时间，但会使存储的文件变得巨大
rdbcompression yes

# 设置是否进行 RDB 文件格式校验，该校验过程在写文件和读文件过程均进行
# 通常默认为开启状态，如果设置为 no，可以节约读写性过程约 10% 时间消耗，但是有一定的数据损坏风险
rdbchecksum yes
```

##### save 指令工作原理

![UTOOLS1576549197074.png](https://img02.sogoucdn.com/app/a/100520146/9fef34f51a489bf5db55d5c323ac0b0d)

![UTOOLS1576549256339.png](https://img02.sogoucdn.com/app/a/100520146/a5a92ab6aa46b4b824ce14f3e7643acc)

#### bgsave 指令

```shell
# 手动启动后台保存操作，但不是立即执行
bgsave

## bgsave 指令相关配置

# 后台存储过程中如果出现错误现象，是否停止保存操作，通常默认为开启状态
stop-writes-on-bgsave-error yes
```

![UTOOLS1576549351431.png](https://img04.sogoucdn.com/app/a/100520146/61d42c371f7e987505f1bc54baf7967d)

#### 后台 save 配置

```shell
# Redis 服务器自动执行指令

# 满足限定时间范围内 key 的变化数量达到指定数量即进行持久化
# second 监控时间范围，changes 监控 key 的变化量
save second changes
```

##### Save 配置原理

![UTOOLS1576549727101.png](https://img04.sogoucdn.com/app/a/100520146/b610c889d7b827372cb8053f212be5a9)

##### RDB 启动方式对比

![UTOOLS1576549774347.png](https://img01.sogoucdn.com/app/a/100520146/f90a86f88e1b7995fe30692e46cea7cc)

#### RDB 特殊启动形式

```shell
# 全量复制

# 服务器运行过程中重启
debug reload

# 关闭服务器时指定保存数据
shutdown save

# 默认情况下执行 shutdown 命令时，自动执行 bgsave（如果没有开启 AOF 持久化功能）
```

#### RDB 优缺点分析

```shell
# RDB 优点:
	# RDB 是一个紧凑压缩的二进制文件，存储效率较高
	# RDB 内部存储的是 redis 在某个时间点的数据快照，非常适合用于数据备份，全量复制等场景
	# RDB 恢复数据的速度要比 AOF 快很多
	# 应用场景: 服务器中每 X 小时执行 bgsave 备份，并将 RDB 文件拷贝到远程机器中，用于灾难恢复

# RDB 缺点:
	# RDB 方式无论是执行指令还是利用配置，无法做到实时持久性，具有较大的可能性丢失数据
	# bgsave 指令每次运行要执行 fork 操作创建子进程，要牺牲掉一些性能
	# Redis 的众多版本中未进行 RDB 文件格式的版本统一,有可能出现各版本服务器之间数据格式无法兼容现象
```

### AOF

#### RDB 存储的弊端

```shell
# 存储数据量较大，效率较低
	# 基于快照思想，每次读写都是全部数据，当数据量巨大时，效率非常低
	
# 大数据量下的 IO 性能较低

# 基于 fork 创建子线程，内存产生额外消耗

# 宕机带来的数据丢失风险
```

##### 解决思路

```shell
# 不写全数据，仅记录部分数据

# 降低区分数据是否改变的难度，改记录数据为记录操作过程

# 读所有操作均进行记录，排除丢失数据的风险
```

#### AOF 概念

```shell
# AOF 持久化: 以独立日志的方式记录每次写命令，重启时再重新执行 AOF 文件中命令达到恢复数据的目的。
	# 与 RDB 相比可以简单描述为 改记录数据为记录数据产生的过程
	
# AOF 的主要作用是解决了数据持久化的实时性，目前已经是 Redis 持久化的主流方式
```

#### AOF 写数据过程

![UTOOLS1576550345678.png](https://img03.sogoucdn.com/app/a/100520146/077b70d68586a9babb65887191647114)

#### AOF 写数据三种策略

```shell
# always
	# 每次写入操作均同步到 AOF 文件中，数据零误差，性能较低
	
# everysec
	# 每秒将缓冲区中的指令同步到 AOF 文件中，数据准确性较高，性能较高
	# 在系统突然宕机的情况下，丢失 1秒内的数据
	
# no
	# 由操作系统控制每次同步到 AOF 文件的周期，整体过程不可控
```

#### AOF 功能开启及相关配置

```shell
# 是否开启 AOF 持久化功能，默认为不开启
appendonly yes|no

# AOF 写数据策略
appendfsync always|everysec|no

# AOF 持久化文件名，默认 appendonly.aof，建议配置为 appendonly-端口号.aof
appendfilename filename

# AOF 持久化文件保存路径，与 RDB 持久化文件保持一致即可
dir
```

#### AOF 写数据遇到的问题

![UTOOLS1576550627141.png](https://img03.sogoucdn.com/app/a/100520146/9d89a86cbe21fac4777be9e7015cc510)

##### AOF 重写

```shell
# 随着命令不断写入 AOF，文件会越来越大
	# 为了解决这个问题，Redis 引入了 AOF 重写机制压缩文件体积。
	# AOF 文件重写是将 Redis 进程内的数据转化为写命令同步到新 AOF 文件的过程。
	# 简单说，对同一个数据的若干个指令执行结果转化成最终结果数据对应的指令进行记录。
	
# 作用
	# 降低磁盘占用量，提高磁盘利用率
	# 提高持久化效率，降低持久化写时间，提高 IO 性能
	# 降低数据恢复用时，提高数据恢复效率
	
# 规则
	# 进程内已超时的数据不再写入文件
	# 忽略无效指令，重写时使用进程内数据直接生成，这样新的 AOF 文件只保留最终数据的写入命令
	# 对同一数据的多条写命令合并为一条命令
		# 为防止数据量过大造成客户端缓冲区溢出，对 list、set、hash、zset 等类型，每条指令最多写入64个元素
```

##### AOF 重写方式

```shell
# 手动重写
bgrewriteaof

# 自动重写
auto-aof-rewrite-min-size size
auto-aof-rewrite-percentage percentage
```

##### AOF 自动重写方式

![UTOOLS1576551237963.png](https://img03.sogoucdn.com/app/a/100520146/8d250df08ce08864a6962416ef8c50ec)

##### AOF 工作流程

![image-20191217105427366](/Users/apple/Library/Application Support/typora-user-images/image-20191217105427366.png)

##### AOF 重写流程

![UTOOLS1576551286000.png](https://img02.sogoucdn.com/app/a/100520146/b2b337d2c06ed6aea946b30b95a42b51)

![UTOOLS1576551430100.png](https://img01.sogoucdn.com/app/a/100520146/aa6d3761decf33ff5685049789cc0d1a)

### RDB VS AOF

![UTOOLS1576551468233.png](https://img03.sogoucdn.com/app/a/100520146/58550f98198b79c7706601b2dc958335)

### RDB 与 AOF 的选择

```shell
# 对数据非常敏感，建议使用默认的 AOF 持久化方案
	# 策略建议使用 everysceond
	# 注意: 由于 AOF 文件存储体积较大，且恢复速度较慢
	
# 数据呈现阶段有效性，建议使用 RDB 持久化方案
	# 数据可以良好的做到阶段内无丢失，恢复速度较快
	
# 综合:
	# 业务数据非常敏感，使用 AOF
	# 能承受分钟以内的数据丢失，且追求大数据集的恢复速度，选用 RDB
	# 灾难恢复选用 RDB
	# 双保险策略，同时开启 RDB 和 AOF
```

## Redis 事务

### 什么是事务

```shell
# Redis 在执行指令过程中，多条连续执行的指令被干扰，打断，插队

# Redis 事务就是一个命令执行的队列，将一系列预定命令包装成一个整体（一个队列）
	# 执行时，中间不能被打断或干扰
```

### 事务的基本操作

```shell
# 开启事务，次指令执行后，后续所有指令均加入到事务中
multi

# 设定事务的结束位置，同时执行事务。与 multi 成对出现，成对使用
exec

# 注意: 加入事务的命令暂时进入到任务队列中，并没有立即执行，只有执行 exec 命令才开始执行

# 终止当前事务的定义，发生在 multi 之后，exec 之前
```

### 事务的工作流程

![UTOOLS1576553218841.png](https://img04.sogoucdn.com/app/a/100520146/5ce7e2df09645c5d8dc848d44bbeea2f)

### 事务的注意事项

```shell
# 定义事务的过程中，命令格式输入错误怎么办？
	# 语法错误:
		# 指命令书写格式有误
		
	# 处理结果:
		# 如果定义的事务中所包含的命令存在语法错误，整体事务中所有命令均不会执行
		
# 定义事务的过程中，命令执行出现错误怎么办？
	# 运行错误:
		# 指命令格式正确，但是无法正确的执行。例如对 list 进行 incr 操作
		
	# 处理结果:
		# 能够正确运行的命令会执行，运行错误的命令不会被执行
		
# 注意: 已经执行完毕的命令对应的数据不会自动回滚，需要程序员自己在代码中实现回滚
```

#### 手动进行事务回滚

```shell
# 记录操作过程中被影响的数据之前的状态
	# 单数据: String
	# 多数据: hash、list、set、zset
	
# 记录指令恢复所有的被修改的项
	# 单数据: 直接 set(注意周边属性，例如时效)
	# 多数据: 修改对应值或整体克隆复制
```

### 基于特定条件的事务执行 - 锁

```shell
# 业务分析:
	# 多个客户端可能同时操作同一组数据，并且该数据一旦被操作修改后，将不适用于继续操作
	# 在操作之前锁定要操作的数据，一旦发生变化，终止当前操作
	
# 解决方案:
	# 对 key 添加监视锁，在执行 exec 前如果 key 发生了变化，终止事务执行
	watch key1 [key2...]
	
	# 取消对所有 key 的监视
	unwatch
	
# Tips 18:
	# Redis 应用基于状态控制的批量任务执行
```

### 基于特定条件的事务执行 - 分布式锁

```shell
# 业务分析:
	# 虽然 redis 是单线程的，但是多个客户端对同一数据进行操作时，如何避免不被同时修改？
	
# 解决方案:
	# 使用 setnx 设置一个公共锁
	setnx lock-key value
	
	# 利用 setnx 命令的返回值特征，有值则返回设置失败，无值则返回设置成功
		# 对于返回设置成功的，拥有控制权，进行下一步的具体业务操作
		# 对于返回设置失败的，不具有控制权，排队或等待
		# 操作完毕通过 del 操作释放锁
		
	# 注意:
		# 上述解决方案是一种设计概念，依赖规范保障，具有风险性
		
# Tips 19:
	# Redis 应用基于分布式锁对应的场景控制
```

### 基于特定条件的事务执行 - 分布式锁改良

```shell
# 业务场景:
	# 依赖分布式锁的机制，某个用户操作时对应客户端宕机，且此时已经获取到锁，如果解决？
	
# 业务分析:
	# 由于锁操作由用户控制加锁解锁，必定会存在加锁后未解锁的风险
	# 需要解锁操作不能仅依赖用户控制，系统级别要给出对应的保底处理方案
	
# 解决方案:
	# 使用 expire 为锁 key 添加时间限定，到时不释放，放弃锁
	expire lock-key second
	pexpire lock-key milliseconds
	
	# 由于操作通常都是微妙或毫秒级，因此该锁定时间不宜设置过大
```

## Redis 删除策略

### 过期数据

#### Redis 中的数据特征

```shell
# Redis 是一种内存级数据库，所有数据均存放在内存中，内存中的数据可以通过 TTL 指令获取其状态
	# XX : 具有时效性的数据
	# -1 : 永久有效的数据
	# -2 : 已经过期的数据 或 被删除的数据 或 未定义的数据
```

### 数据删除策略

```shell
# 定时删除

# 惰性删除

# 定期删除
```

#### 时效性数据的存储结构

![UTOOLS1576560159228.png](https://img02.sogoucdn.com/app/a/100520146/fd522f67852de6d2a6e92ac2fd2138b2)

#### 数据删除策略的目标

```shell
# 在内存占用与 CPU 占用之间寻找一种平衡
	# 顾此失彼都会造成整体 redis 性能的下降，甚至引发服务器宕机或内存泄漏
```

##### 定时删除

```shell
# 创建一个定时器，当 key 设置有过期时间，且当过期时间到达时，由定时器任务立即执行对键的删除操作

# 优点: 节约内存，到时就删除，快速释放掉不必要的内存占用

# 缺点: CPU 压力很大，无论 CPU 此时负载量多高，均占用 CPU，会影响 Redis 服务器响应时间和指令吞吐量

# 总结: 用处理器性能换存储空间（时间换空间）
```

##### 惰性删除

```shell
# 数据到达过期时间，不做处理，等下次访问该数据时:
	# 如果未过期，返回数据
	# 发现已过期，删除，返回不存在
	
# 优点: 节约 CPU 性能，发现必须删除的时候才删除

# 缺点: 内存压力很大，出现长期占用内存的数据

# 总结: 用存储空间换取处理器性能（空间换时间）
```

##### 定期删除

![UTOOLS1576560507471.png](https://img02.sogoucdn.com/app/a/100520146/c25c4ec6243e517bf0943392f076b0d4)

```shell
# 周期性轮询 redis 库中的时效性数据，采用随机抽取的策略，利用过期数据占比的方式控制删除频度

# 特点1: CPU 性能占用设置有峰值，检测频度可自定义设置

# 特点2: 内存压力不是很大，长期占用内存的冷数据会被持续清理

# 总结: 周期性抽查存储空间（随机抽查，重点抽查）
```

#### 删除策略对比

![UTOOLS1576560622445.png](https://img03.sogoucdn.com/app/a/100520146/8491bf9ff93e72e699eab7f98461fc7b)

### 逐出算法

#### 新数据进入检测

```shell
# 当新数据进入 Redis 时，如果内存不足怎么办？
	# Redis 使用内存存储数据，在执行每一个命令前，会调用 freeMemoryIfNeeded() 检测内存是否充足
	# 如果内存不满足新加入数据的最低存储要求，Redis 要临时删除一些数据为当前指令清理存储空间
	# 清理数据的策略称为 逐出算法
	
# 注意: 逐出数据的过程不是 100% 能够清理出足够的可使用的内存空间，如果不成功则反复执行。
	# 当对所有数据尝试完毕后，仍不能达到内存清理要求，出现错误信息:
```

![UTOOLS1576560803137.png](https://img01.sogoucdn.com/app/a/100520146/a361d365cf0bad96459609f136fbb24e)

#### 逐出策略配置

```shell
# 最大可使用内存
# 占用物理内存的比例，默认值为0，表示不限制，生产环境中通常设置在 50% 以上
maxmemory

# 每次选取待删除数据的个数
# 选取数据时并不会全库扫描，导致严重的性能消耗，降低读写性能。因此采用随机获取数据的方式作为待检测删除数据
maxmemory-samples

# 删除策略
# 到达最大内存后，对被挑选出来的数据进行删除的策略
maxmemory-policy


# 检测易失数据（可能会过期的数据集 server.db[i].expires）
	# volatile-lru : 挑选最近最少使用的数据淘汰
	
	# volatile-lfu : 挑选最近使用次数最少的数据淘汰
	
	# volatile-ttl : 挑选将要过期的数据淘汰
	
	# volatile-random : 任意选择数据淘汰
	
# 检测全库数据（所有数据集 server.db[i].dict）
	# allkeys-lru
	
	# allkeys-lfu
	
	# allkeys-random
	
# 放弃数据驱逐
	# no-enviction : 禁止驱逐数据（Redis4.0 中默认策略，会引发错误 OOM）
```

#### 数据逐出策略配置依据

```shell
# 使用 INFO 命令输出监控信息，查询缓存 hit 和 miss 的次数，根据业务需求调优 Redis 配置
```

## Redis 服务器配置

### 服务器端设定

```shell
# 设置服务器以守护进程的方式运行
daemonize yes|no

# 绑定主机地址
bind 127.0.0.1

# 设置服务器端口号
port 6379

# 设置数据库数量
database 16
```

### 日志配置

```shell
# 设置服务器以指定日志记录级别
loglevel debug|verbose|notice|warning

# 日志记录文件名
logfile 端口号.log

# 注意: 日志级别开发期设置为 verbose 即可，生产环境中配置为 notice，简化日志输出
```

### 客户端配置

```shell
# 设置同一时间最大客户端连接数，默认无限制，当客户端连接到达上线，Redis 会关闭新的连接
maxclients 0

# 客户端限制等待最大时长，达到最大值后关闭连接，如需关闭该功能，设置为0
timeout 300
```

### 多服务器快捷配置

```shell
# 导入并加载指定配置文件信息，用于快速创建 redis 公共配置较多的 redis 实例配置文件，便于维护
include /path/server-端口号.conf
```

## Redis 高级数据类型

### Bitmaps

#### 存储需求

![UTOOLS1576564013980.png](https://img03.sogoucdn.com/app/a/100520146/c63c0959e1feb79d684a1b4becd3d0ba)

#### Bitmaps 类型的基础操作

```shell
# 获取指定 key 对应偏移量上的 bit 值
getbit key offset

# 设置指定 key 对应偏移量上的 bit 值，value 只能是 1 或 0
setbit key offset value
```

#### Bitmaps 类型的扩展操作

```shell
# 业务场景
	# 电影网站
		# 统计每天某一部电影是否被点播
		# 统计每天有多少部电影被点播
		# 统计每周/月/年 有多少部电影被点播
		# 统计年度哪部电影没有被点播
		
# 业务分析:
```

![UTOOLS1576564347928.png](https://img02.sogoucdn.com/app/a/100520146/5c1a45de5af20abe9e8d713a6cdced69)

```shell
# 对指定 key 按位进行交、并、非、异或操作，并将结果保存到 destKey 中
bitop op destKey key1 [key2...]
	# and: 交
	# or: 并
	# not: 非
	# xor: 异或
	
# 统计指定 key 中 1 的数量
bitcount key [start end]

# Tips 21:
	# Redis 应用与信息状态统计
```

### HyperLogLog

```shell
# 统计独立 UV:
	# 原始方案: set
		# 存储每个用户的 id（字符串）
		
	# 改进方案: Bitmaps
		# 存储每个用户状态（bit）
		
	# 全新的方案: Hyperloglog
	
# 基数:
	# 基数是数据集去重后元素个数
	# HyperLogLog 是用来做基数统计的，运用了 LogLog 的算法
```

![UTOOLS1576564690735.png](https://img03.sogoucdn.com/app/a/100520146/02064ca7aedbe59eb43bc550fde5c80e)

#### HyperLogLog 类型的基本操作

```shell
# 添加数据
pfadd key element [element...]

# 统计数据
pfcount key [key ...]

# 合并数据
pfmerge destkey sourcekey [sourcekey...]

# Tips 22:
	# redis 应用于独立信息统计
```

#### 相关说明

```shell
# 用于进行基数统计，不是集合，不保存数据，只记录数量而不是具体数据

# 核心是基数估算算法，最终数值存在一定误差

# 误差范围: 基数估计的结果是一个带有 0.81% 标准错误的近似值

# 耗空间极小，每个 hyperloglog key 占用了12k 的内存用于标记基数

# pfadd 命令不是一次性分配 12K 内存使用，会随着基数的增加内存逐渐增大

# pfmerge 命令合并后占用的存储空间为 12K，无论合并之前数据量多少
```

### GEO

#### GEO 类型的基本操作

```shell
# 添加坐标点
geoadd key longitude latitude member [longitude latitude member ...]

# 获取坐标点
geopos key member [member ...]

# 计算坐标点距离
geodist key member1 member2 [unit]

# 添加坐标点
georadius key longitude latitude redius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]

# 获取坐标点
georadiusbymember key member redius m|km|ft|mi [withcoord] [withdist] [withhash] [count count]

# 计算经纬度
geohash key member [member ...]

# Tips 23:
	# Redis 应用于地理位置计算
```

## 企业级解决方案

### 缓存预热

```shell
# 业务场景:
	# 服务器启动后迅速宕机:

# 问题排查:
	# 请求数量较多
	# 主从之间数据吞吐量较大，数据同步操作频度较高
	
# 解决方案:
	# 前置准备工作:
		# 日常例行统计数据访问记录，统计访问频度较高的热点数据
		# 利用 LRU 数据删除策略，构建数据留存队列
			# 例如: storm + kafka
			
	# 准备工作:
		# 将统计结果中的数据分类，根据级别，redis 优先加载级别较高的热点数据
		# 利用分布式多服务器同时进行数据读取，提速数据加载过程
		# 热点数据主从同时预热
		
	# 实施:
		# 使用脚本程序固定触发数据预热过程
		# 如果条件允许，使用了 CDN（内容分发网络），效果会更好
		
	# 总结:
		# 缓存预热就是系统启动前，提前将相关的缓存数据直接加载到缓存系统。
		# 避免在用户请求的时候，先查询数据库，然后再讲数据缓存的问题
		# 用户直接查询事先被预热的缓存数据
```

### 缓存雪崩

```shell
# 业务场景:
	# 系统平稳运行过程中，忽然数据库连接量激增
	# 应用服务器无法及时处理请求
	# 大量 408, 500 错误页面出现
	# 客户反复刷新页面获取数据
	# 数据库崩溃
	# 应用服务器崩溃
	# 重启应用服务器失效
	# Redis 服务器崩溃
	# Redis 集群崩溃
	# 重启数据库后再次被瞬间流量放倒
	
# 问题排查:
	# 在一个较短的时间内，缓存中较多的 key 集中过期
	# 此周期内请求访问过期的数据，redis 未命中，redis 向数据库获取数据
	# 数据库同时接收到大量的请求无法及时处理
	# Redis 大量请求被积压，开始出现超时现象
	# 数据库流量激增，数据库崩溃
	# 重启后仍然面对缓存中无数据可用
	# Redis 服务器资源被严重占用，Redis 服务器崩溃
	# Redis 集群呈现崩溃，集群瓦解
	# 应用服务器无法及时得到数据响应请求，来自客户端的请求数量越来越多，应用服务器崩溃
	# 应用服务器，redis，数据库全部重启，效果不理想
	
# 问题分析:
	# 短时间范围内
	# 大量 key 集中过期
	
# 解决方案（道）:
	# 更多的页面静态化处理
	# 构建多级缓存架构:
		# Nginx 缓存 + Redis 缓存 + ehcache 缓存
		
	# 检测 MySQL 严重耗时业务进行优化
		# 对数据库的瓶颈排查，例如超时时间，耗时较高事务等
		
	# 灾难预警机制:
		# 监控 redis 服务器性能指标
			# CPU 占用、CPU 使用率
			# 内存容量
			# 查询平均响应时间
			# 线程数
			
	# 限流、降级
		# 短时间范围内牺牲一些客户体验，限制一部分请求访问，降低应用服务器压力，待业务低速运转后逐步开放访问
		
# 解决方案（术）
	# LRU 与 LFU 切换
	
	# 数据有效期策略调整
		# 根据业务数据有效期进行分类错峰，A类 90分钟，B类 80分钟，C类 70分钟
		# 过期时间使用固定时间 + 随机值的形式，稀释集中到期的 key 的数量
		
	# 超热数据使用永久 Key
	
	# 定期维护（自动 + 人工）
		# 对即将过期数据做访问量分析，确认是否延时，配合访问量统计，做热点数据的延时
		
	# 加锁
		# 慎用
		
# 总结:
	# 缓存雪崩就是瞬间过期数据量太大，导致对数据库服务器造成压力
	# 如能有效避免过期时间集中，可以有效解决雪崩现象的出现（约 40%）
	# 配合其他策略一起使用，并监控服务器的运行数据，根据运行记录做快速调整
```

### 缓存击穿

```shell
# 业务场景:
	# 系统平稳运行过程中
	# 数据库连接量瞬间激增
	# Redis 服务器无大量 key 过期
	# Redis 内存平稳，无波动
	# Redis 服务器 CPU 正常
	# 数据库崩溃
	
# 问题排查:
	# Redis 中某个 key 过期，该 key 访问量巨大
	# 多个数据请求从服务器直接压到 Redis 后，均未命中
	# Redis 在短时间内发起了大量对数据库中同一数据的访问
	
# 问题分析:
	# 单个 Key 高热数据
	# key 过期
	
# 解决方案（术）:
	# 预先设定:
		# 以电商为例，每个商家根据店铺等级，指定若干款主打商品，在购物节期间，加大此类信息 Key 的过期时长
		# 注意: 购物节不仅仅指当天，以及后续若干天，访问峰值呈现逐渐降低的趋势
		
	# 现场调整:
		# 监控访问量，对自然流量激增的数据延长过期时间或设置为永久性 Key
		
	# 后台刷新数据:
		# 启动定时任务，高峰期来临之前，刷新数据有效期，确保不丢失
		
	# 二级缓存:
		# 设置不同的失效时间，保障不会被同时淘汰就行
		
	# 加锁:
		# 分布式锁，防止被击穿，但是要注意也是性能瓶颈，慎重!
		
# 总结:
	# 缓存击穿就是单个高热数据过期的瞬间，数据访问量较大
	# 未命中 redis 后，发起了大量对同一数据的数据库访问，导致对数据库服务器造成压力
	# 应对策略应该在业务数据分析与预防方面进行，配合运行监控测试与及时调整策略
	# 毕竟单个 Key 的过期监控难度较高，配合雪崩处理策略即可
```

### 缓存穿透

```shell
# 业务场景:
	# 系统平稳运行过程中
	# 应用服务器流量随时间增量较大
	# Redis 服务器命中率随时间逐步较低
	# Redis 内存平稳，内存无压力
	# Redis 服务器 CPU 占用激增
	# 数据库服务器压力激增
	# 数据库崩溃
	
# 问题排查:
	# Redis 中大面积出现未命中
	# 出现非正常 URL 访问
	
# 问题分析:
	# 获取的数据在数据库中也不存在，数据库查询未得到对应数据
	# Redis 获取到 null 数据未进行持久化，直接返回
	# 下次此类数据到达重复上述过程
	# 出现黑客攻击服务器
	
# 解决方案（术）:
	# 缓存 null
		# 对查询结果为 null 的数据进行缓存（长期使用，定期清理），设定短时限，例如 30 - 60秒，最高5分钟
		
	# 白名单策略:
		# 提前预热各种分类数据 id 对应的 bitmaps，id 作为 bitmaps 的 offset，相当于设置了数据白名单
		# 当加载正常数据时，放行，加载异常数据时直接拦截（效率偏低）
		# 使用布隆过滤器（有关布隆过滤器的命中问题对当前状况可以忽略）
		
	# 实施监控:
		# 实时监控 Redis 命中率（业务正常范围时，通常会有一个波动值）与null 数据的占比
			# 非活动时段波动: 通常检测 3-5 倍，超过 5 倍纳入重点排查对象
			# 活动时段波动: 通常检测 10-50 倍，超过 50 倍纳入重点排查对象
		# 根据倍数不同，启动不同的排查流程，然后使用黑名单进行防控（运营）
		
	# key 加密:
		# 问题出现后，临时启动防灾业务 key，对 key 进行业务层传输加密服务，设定校验程序，过来的 key 校验
		# 例如每天随机分配 60 个加密串，挑选 2-3 个，混淆到页面数据 id 中
			# 发现访问 key 不满足规则，驳回数据即可
		
# 总结:
	# 缓存击穿访问了不存在的数据，跳过了合法数据的 redis 数据缓存阶段
	# 每次访问数据库，导致对数据库服务器造成压力。
	# 通常此类数据的出现量是一个较低的值，当出现此类情况以毒攻毒，并及时报警
	# 无论是黑名单还是白名单，都是对整体系统的压力，警报解除后尽快移除
```

### 性能指标监控

#### 性能指标 Performance

| Name                      | Description              |
| ------------------------- | ------------------------ |
| latency                   | Redis 响应一个请求的时间 |
| instantaneous_ops_per_sec | 平均每秒处理请求总数     |
| hit rate（calculated）    | 缓存命中率（计算出来的） |

#### 内存指标 Memory

| Name                    | Description                                    |
| ----------------------- | ---------------------------------------------- |
| user_memory             | 已使用内存                                     |
| mem_fragmentation_ratio | 内存碎片率                                     |
| evicted_keys            | 由于最大内存限制被移除的 key 的数量            |
| blocked_clients         | 由于 BLPOP、BRPOP、BRPOPLPUSH 而被阻塞的客户端 |

#### 基本活动指标 Basic activity

| Name                       | Description                |
| -------------------------- | -------------------------- |
| connected_clients          | 客户端连接数               |
| connected_slaves           | Slave 数量                 |
| master_last_io_seconds_ago | 最近一次主从交互之后的秒数 |
| keyspace                   | 数据库中的key 值总数       |

#### 持久性指标 Persistence

| Name                        | Description                        |
| --------------------------- | ---------------------------------- |
| rdb_last_save_time          | 最后一次持久化保存到磁盘的时间戳   |
| rdb_changes_since_last_save | 自最后一次持久化以来数据库的更改数 |

#### 错误指标 Error

| Name                           | Description                             |
| ------------------------------ | --------------------------------------- |
| rejected_connections           | 由于达到 maxclient 限制而被拒绝的连接数 |
| keyspace_misses                | Key 值查找失败（没有命中）次数          |
| master_link_down_since_seconds | 主从断开的持续时间（以秒为单位）        |

#### 监控方式

##### 工具

```shell
# Cloud Insight Redis

# Prometheus

# Redis-stat

# Redis-faina

# RedisLive

# zabbix
```

##### 命令

```shell
# benchmark

# redis cli
	# monitor
	# showlog
```

![UTOOLS1576634336786.png](https://img01.sogoucdn.com/app/a/100520146/cdac93863b78f12df2e6bdbdd9a12aa3)

![UTOOLS1576634351811.png](https://img03.sogoucdn.com/app/a/100520146/07ac7ed52dbf6ae16cda1c170bbf3bad)

![UTOOLS1576634369200.png](https://img02.sogoucdn.com/app/a/100520146/0faeb4c897a93541c70b9b66754c1c8b)

