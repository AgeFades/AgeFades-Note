# 鲁班 - Zookeeper

## zk 概要、产生背景、作用

```shell
# 产生背景:
	# 项目从单体转到分布式，产生多个节点之间协同的问题，例如:
		# 定时任务由哪个节点执行...
		# RPC 调用时的服务注册与发现中心...
		# 如果保证并发请求的幂等...
		
	# 这些问题统一归纳为多节点协调问题，如果靠节点自身是不可靠的。
	
	# 所以 zk 诞生，一个独立服务进行协调工作。
	
# 概要:
	# 用于分布式应用程序的协调服务。
	
	# 公开一组简单的API，分布式App 可以基于这些Api 进行同步、节点状态、配置、服务注册...
	
	# 由 Java 编写，支持 Java 和 C 两种语言的客户端。
```

### znode 节点

![图片](https://uploader.shimo.im/f/vmq8ZNmrsWk7v199.png!thumbnail)

```shell
# zk 中的数据基本单元叫节点。

# 节点之下可包含子节点，最终以树的形式呈现。

# 每个节点拥有唯一的路径 path。

# 客户端基于 path 上传节点数据，zk 收到后实时通知对该路径监听的客户端。
```

## 部署与常规配置

```shell
# zk 需要 JVM 环境运行，默认端口 2181。
```

### 版本说明

```shell
# 3.5.5 被认为是 3.4 稳定分支的后续版本，可用于生产。

# 对比 3.4 包含以下新功能:
	# 动态重新配置
	# 本地会议
	# 新节点类型: 容器、ttl
	# 原子广播协议的 SSL 支持
	# 删除观察者的能力
	# 多线程提交处理器
	# 升级到 Netty4.1
	# Maven 构建
	
# 注意: 建议最低 JDK 版本为 1.8
```

### 下载安装

```shell
# 下载地址 
https://zookeeper.apache.org/releases.html#download
```

```shell
# 安装流程

# 此处使用 Mac 网页下载压缩包安装，Linux 参考如下命令
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/current/apache-zookeeper-3.5.5-bin.tar.gz

# 以下流程不区分 OS
tar -zxvf apache-zookeeper-3.5.5-bin.tar.gz

# ${zk_home} 表示你解压文件所在目录
cd ${zk_home}/conf

# 复制一份配置文件
cp zoo_sample.cfg zoo.cfg

# 启动
{zk_home}/bin/zkServer.sh start

# 连接客户端
{zk_home}/bin/zkCli.sh
```

### 常规配置

![UTOOLS1584518952331.png](http://yanxuan.nosdn.127.net/466263c2ad4bc5eabf24c75181d7d618.png)

```shell
# zk 时间配置中的基本单位，心跳时间（毫秒），默认为 2 秒
tickTime=2000

# 允许集群中 follower 初始化连接到 leader 的最大时长
# 表示的是 tickTime 时间倍数，即: initLimit * tickTime
initLimit=10

# 允许集群中 follower 与 leader 数据同步的最大时长
# 表示 tickTime 时间倍数
syncLimit=5

# zk 数据存储目录
dataDir=/tmp/zookeeper

# 对客户端提供的端口号
clientPort=2181

# 单个客户端与 zk 最大并发连接数
maxClientCnxns=60

# 保存的数据快照数量，之外的将会被清除
autopurge.snapRetainCount=3

# 自动触发清除任务时间间隔，小时为单位，默认为0，表示不自动清除
autopurge.purgeInterval=1
```

### 客户端命令

```shell
# 关闭当前会话
close

# 重新连接指定 zk 服务
connect host:port

# 创建节点
create [-s] [-e] [-c] [-t ttl] path [data] [acl]

# 删除节点（不能存在子节点）
delete [-v version] path

# 删除路径及所有子节点，3.6.0 版本变为 rmr 命令
deleteall path

# 设置节点限额 -n 子节点数 -b 字节数
setquota -n|-b val path

# 查看节点限额
listquota path

# 删除节点限额
delquota [-n|-b] path

# 查看节点数据 -s 包含节点状态 -w 添加监听
get [-s] [-w] path

# 获取权限配置
getAcl [-s] path

# 列出子节点 -s 状态 -R 递归查看所有子节点 -w 添加监听
ls [-s] [-w] [-R] path

# 是否打印监听事件
printwatches on|off

# 退出客户端
quit

# 查看执行的历史记录
history

# 重复执行命令，history 中命令编号确定
redo [cmdno]

# 删除指定监听
removewatches path [-c|-d|-a] [-l]

# 设置值
set [-s] [-v version] path data

# 为节点设置 ACL 权限
setAcl [-s] [-v version] [-R] path acl

# 查看节点状态 -w 添加监听
stat [-w] path

# 强制节点同步
sync path
```

### 命令Demo

```shell
# 查看根路径下所有节点，默认只有 zookeeper 节点
ls /
```

![UTOOLS1584600664527.png](http://yanxuan.nosdn.127.net/818ff4f87f67844052f243ee0625e92f.png)

```shell
# 创建节点，必须以 / 开头，绝对路径
create /["节点路径"] ["节点数据"]
```

![UTOOLS1584600939290.png](http://yanxuan.nosdn.127.net/a9ef4b8a80bac99f483d0fe0daad980d.png)

```shell
# 获取节点信息
get /["节点路径"]
```

![UTOOLS1584601190471.png](http://yanxuan.nosdn.127.net/93ded511c476001968a9b08e5e8bf40d.png)

```shell
# 删除节点
delete /["节点路径"]
```

![UTOOLS1584601227582.png](http://yanxuan.nosdn.127.net/8158bb3907d4cdc6967a9a3153a26f98.png)

```shell
# 修改节点
set /["节点路径"] ["节点数据"]
```

![UTOOLS1584601384291.png](http://yanxuan.nosdn.127.net/f0b19056325ff16643136550c195ec2e.png)

## zk 节点介绍

### znode

```shell
# zk 中节点叫 znode。

# 存储结构上跟文件系统类似，以树级结构进行存储。

# 没有目录的概念，没有 cd 等命令。

# znode 的结构:
	# path : 唯一路径
	# childNode : 子节点
	# stat : 状态属性
	# type : 节点类型
```

### 节点类型

#### 持久节点

```shell
# persistent : 持久化节点

# 默认创建的就是持久节点
create /["节点路径"]
```

#### 持久序号节点

```shell
# persistent_sequential : 持久序号节点

# 创建时 zk 会在路径上加上序号作为后缀。

# 非常适用于分布式锁、分布式选举等场景。

# 创建时添加 -s 参数即可
```

![UTOOLS1584602347914.png](http://yanxuan.nosdn.127.net/beb10df6125e0c5afd42f99a4e51cb86.png)

#### 临时节点

```shell
# ephemeral : 临时节点

# 临时节点会在客户端会话断开后自动删除。

# 适用于心跳，服务发现等场景。

# 创建时添加参数 -e 即可

# 注意: 在 3.5.5 版本及以前，临时节点下不能有子节点，3.6.0 之后可以。
```

![UTOOLS1584604127434.png](http://yanxuan.nosdn.127.net/4344e651958644b19cb71490ef123df7.png)

![UTOOLS1584606071089.png](http://yanxuan.nosdn.127.net/82cc913b9981172244ad9540e44ab26a.png)

#### 临时序号节点

```shell
# ephemeral_sequential : 临时节点

# 与持久序号节点类似，不同在于会话断开后就会删除。

# 创建时添加参数 —e -s 即可
```

![UTOOLS1584604340866.png](http://yanxuan.nosdn.127.net/01698495c3fa9ee3ff48fb395c5fc27f.png)

### 节点属性

```shell
# 查看节点属性
stat /["节点路径"]
```

![UTOOLS1584604451861.png](http://yanxuan.nosdn.127.net/b6344e2f4054734cc887af591cc5c4e4.png)

```shell
# cZxid : 创建节点的事务ID

# ctime : 创建时间

# mZxid : 修改节点的事务ID

# mtime : 最后修改时间

# pZxid : 子节点变更的事物ID

# cversion : 表示对此 znode 的子节点进行的更改次数（不包含更下级别的子节点更改次数）

# dateVersion : 数据变更次数

# aclVersion : 权限变更次数

# ephemeralOwner : 临时节点所属会话ID， OxO 为持久节点

# dataLength : 数据长度

# numChildren : 子节点数（不包括更下级别的子节点数）
```

### 节点的监听

```shell
# 客户添加 -w 参数可实时监听节点与子节点的变化，并且实时收到通知。

# 监听是一次性的，触发一次过后就会作废。

# 非常适用保障分布式情况下的数据一致性。
```

#### 监听子节点

```shell
# 监听子节点的变化（增、删）
ls ["节点路径"] watch
```

![UTOOLS1584609533824.png](http://yanxuan.nosdn.127.net/66168f88b5e19f4d94f5fdb1168b6b2c.png)

![UTOOLS1584609513588.png](http://yanxuan.nosdn.127.net/aa92b69ed0fd2a9224d5ddbbf41c8905.png)

#### 监听节点数据

```shell
# 监听节点数据的变化
get ["节点路径"] watch
```

![UTOOLS1584609204946.png](http://yanxuan.nosdn.127.net/87c0e08d662ed1f73133ac3f8108295b.png)

![UTOOLS1584609220542.png](http://yanxuan.nosdn.127.net/13789d0051ad76274d68ad5080bbff75.png)

#### 监听节点状态

```shell
# 监听节点状态的变化
stat ["节点路径"] watch
```

### acl 权限设置

```shell
# Access Control List : 访问控制列表

# 用于控制资源的访问权限。

# zk 使用 acl 来控制对其 znode 的访问。

# 基于 scheme:id:permission 的方式进行权限控制。
	# scheme 表示授权模式
	# id 模式对应值
	# permission 即具体的增删改权限位
```

#### scheme 认证模型

```shell
# world : 开放，全可以访问（默认设置）。

# ip : 限定客户端 ip 访问。

# auth : 用户密码认证模式，只有在会话中添加了认证才可以访问，明文密码。

# digest : 与 auth 类似，用 sha-1 + base64 加密后的密码，实际使用中 digest 更常见。
```

#### permission 权限位

| 权限位 | 权限   | 描述                             |
| ------ | ------ | -------------------------------- |
| c      | create | 可以创建子节点                   |
| d      | delete | 可以删除子节点（仅下一级节点）   |
| r      | read   | 可以读取节点数据及显示子节点列表 |
| w      | write  | 可以设置节点数据                 |
| a      | admin  | 可以设置节点访问控制列表权限     |

#### acl 相关命令

| 命令    | 使用方式              | 描述          |
| ------- | --------------------- | ------------- |
| getAcl  | getAcl<path>          | 读取 acl 权限 |
| setAcl  | setAcl<path><acl>     | 设置 acl 权限 |
| addauth | addauth<scheme><auth> | 添加认证用户  |

![UTOOLS1584685110104.png](http://yanxuan.nosdn.127.net/140c9f173a1bbc8f46e36e59ebf26a27.png)

![UTOOLS1584685312119.png](http://yanxuan.nosdn.127.net/222a78b3eeb21b04f1727fd60d05ce78.png)

![UTOOLS1584685323614.png](http://yanxuan.nosdn.127.net/38a37a864e1b5a2efadceeb46ea6017a.png)

![UTOOLS1584685432951.png](http://yanxuan.nosdn.127.net/01e94d8ae22d844afc1088639b283eb1.png)

![UTOOLS1584685658676.png](http://yanxuan.nosdn.127.net/ba0986790714ca0446576c5967150e88.png)

## zk 客户端

```shell
# zk 提供了 java 和 c 两种语言的客户端。
```

#### gav

```xml
<dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.5.5</version>
</dependency>
```

#### 初始连接

```shell
# 常规的客户端类是: org.apache.zookeeper.ZooKeeper。

# 实例化该类之后将会自动与集群建立连接。
```

```shell
# 构造参数:

# connectString:
	# 连接串，包含 ip + port
	# 集群使用逗号隔开
	# 192.168.0.1:2181,192.168.0.2:2181
	
# sessionTimeout:
	# 会话超时时间
	# 该值不能超过服务端所设置的 minSessionTimeout 和 maxSessionTimeout
	
# watcher:
	# 会话监听器，服务端事件将会触发该监听
	
# sessionId:
	# 自定义会话ID
	
# sessionPasswd:
	# 会话密码
	
# canBeReadOnly:
	# 该连接是否为只读的

# hostProvider:
	# 服务端地址提供者
	# 提示客户端如何选择某个服务来调用
	# 默认采用 StaticHostProvider 实现
```

```java
@Before
public void init() throws Exception {
  String conn = "127.0.0.1:2181";
  Zookeeper zk = new Zookeeper(conn,...);
}
```

#### 创建、查看节点

```shell
# crate() : 创建节点

# getData() : 查看节点

# getChildren() : 查看子节点
```

#### 监听节点

```shell
# getData() 和 getChildren() 可分别设置监听数据变化和子节点变化。

# 通过设置 watch 为 true。

# 当前事件触发时会调用 zk 构建函数中 Watcher.process() 方法。

# 也可以添加 watcher 参数实现自定义监听，一般都是这么干的。

# 注意: 所有的监听都是一次性的，如果要持续监听需要触发后再添加一次监听。
```

#### zkClient

```shell
# zkClient 是在 zk 客户端基础之上封装的。

# 可以设置持久监听、或删除某个监听。

# 可以插入 java 对象，自动进行序列化和反序列化

# 简化了基本的 CRUD
```

```xml
<dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.11</version>
</dependency>
```

## zk 集群

```shell
# zk 集群的目的是为了保证系统的性能、承载更多的客户端连接。

# 集群功能:
	# 读写分离: 提高承载，保证性能。
	# 主从自动切换: 提高服务容错性，避免单点故障。
```

```shell
# 半数以上运行机制说明:

# 集群至少需要三台服务器，并且强烈建议使用奇数个服务器。

# 因为 zk 通过判断大多数节点的存活，来判断整个服务是否可用。

# 3个节点，挂掉两个表示整个集群挂掉，4个也是挂掉两个表示整个集群挂掉。
```

### 集群部署

```shell
# 伪集群搭建，实际中改变 ip，每个实例放入不同机器就好。

# 进入 zk 目录
cd {zk_home}

# 创建文件目录
mkdir data

mkdir data/1

mkdir data/2

mkdir data/3

# 创建 pid 文件
echo 1 > data/1/myid

echo 2 > data/2/myid

echo 3 > data/3/myid

# 编写配置文件
vim conf/zoo1.cfg

--- 配置文件 ---
tickTime=2000
initLimit=10
syncLimit=5
dataDir=data/1
clientPort=2181	# 与客户端通信的端口号
# 集群配置
server.1=127.0.0.1:2887:3887 # 与子节点通信的端口号和选举的端口号
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889
--- 配置文件 ---

# cv zoo1 的配置文件
cp conf/zoo1.cfg conf/zoo2.cfg

cp conf/zoo1.cfg conf/zoo3.cfg

# 修改其他两个节点的配置文件
vim conf/zoo2.cfg

--- 配置文件 ---
tickTime=2000
initLimit=10
syncLimit=5
dataDir=data/2
clientPort=2182
# 集群配置
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889
--- 配置文件 ---

vim conf/zoo3.cfg

--- 配置文件 ---
tickTime=2000
initLimit=10
syncLimit=5
dataDir=data/3
clientPort=2183
# 集群配置
server.1=127.0.0.1:2887:3887
server.2=127.0.0.1:2888:3888
server.3=127.0.0.1:2889:3889
--- 配置文件 ---

# 启动节点，如果只启动一个节点，根据半数以上运行原则，整个集群是不可用的。
./bin/zkServer.sh start conf/zoo1.cfg

./bin/zkServer.sh start conf/zoo2.cfg

./bin/zkServer.sh start conf/zoo3.cfg

# 连接节点，任意选择连接节点数
./bin/zkCli.sh -server 127.0.0.1:2181

./bin/zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182

./bin/zkCli.sh -server 127.0.0.1:2181,127.0.0.1:2182,127.0.0.1:2183

# 测试数据同步

	# 在 2181 节点创建 znode
	create -e -s /test "集群测试"
	
	# 在 2182 节点查看 znode
	ls /

```

![UTOOLS1584931510791.png](http://yanxuan.nosdn.127.net/337f1fa43748884bd3c6c8a1951e516c.png)

![UTOOLS1584932585058.png](http://yanxuan.nosdn.127.net/376a20b437f5352ec38de98332f89726.png)

![UTOOLS1584932596370.png](http://yanxuan.nosdn.127.net/39566df992335d64ad304dd816c6743b.png)

### 集群角色

```shell
# zk 集群共有三种角色。

# leader:
	# 主节点，用于写入数据。
	# 通过选举产生，如果宕机将会选举新的主节点。
	
# follower:
	# 子节点，用于实现数据的读取。
	# 主节点的备选节点，拥有投票权。
	
# observer:
	# 观察节点，用于读取数据。
	# 没有投票权，不能选为主节点。
	# 计算集群可用状态时不会将 observer 计算入内。
```

```shell
# 查看节点角色
./bin/zkServer.sh status conf/zoo1.cfg

./bin/zkServer.sh status conf/zoo2.cfg

./bin/zkServer.sh status conf/zoo3.cfg
```

![UTOOLS1584933589121.png](http://yanxuan.nosdn.127.net/a34f915690e86de14d1c29b082f83078.png)

![UTOOLS1584933657546.png](http://yanxuan.nosdn.127.net/15efa063c7019387435a790663c1d2af.png)

### 选举机制

```shell
# 如果是三个节点新建的集群，第一次 Leader 总是第二个节点。

# 投票机制说明:
	# 第一轮投票全部投给自己
	# 第二轮投票给 myid 比自己大的相邻节点（不是同时，是一个一个的触发）
	# 如果得票超过半数，选举结束
	# 注释（当节点1 投票给节点2 时，此时节点2 已经2票了，自动当选，而不会再投票给节点 3）
```

![图片](https://uploader.shimo.im/f/ChMbWt0DhF4NL7nA.png!thumbnail)

### 选举触发

```shell
# 当集群中出现以下两种情况，就会进行 Leader 的选举。
	# 服务节点初始化完毕。
	
	# 半数以上的节点无法和 Leader 建立连接。
	
	
# 节点初始启动时，会在集群中寻找 Leader。
	# 找到则建立连接，自身状态变为 follower 或 observer。
	
	# 如果没有找到 Leader，当前节点状态将变化 looking，进入选举流程。
	
	# 集群运行期间，follower 宕机只要不超过半数，就不影响集群正常运行。
	
	# 但 leader 宕机，将暂停对外服务，所有 follower 进入 looking，进入选举流程，
```

### 数据同步

```shell
# zk 的数据同步是为了保证各节点中数据的一致性。

# 同步时涉及两个流程:
	# 正常的客户端数据提交。
	
	# 集群某个节点宕机在恢复后的数据同步。
```

#### 客户端写入请求

```shell
# Leader 接收客户端写请求，并同步给各个子节点。
```

![图片](https://uploader.shimo.im/f/k2Dqe4W0OCoumzf3.png!thumbnail)

```shell
# 但实际情况中，客户端可能并不知道哪个节点是 Leader。

# 可能写的请求发给 follower，再由它转发给 Leader 进行同步处理。
```

![图片](https://uploader.shimo.im/f/zQHJd478VV8GoCaK.png!thumbnail)

```shell
# 客户端写入流程说明:

# 1. 客户端向 zk server 发送写请求，如果该 server 不是 leader，则会将该写请求转发给 leader，leader 再将请求事务以 proposal 形式分发给 follower。

# 2. 当 follower 收到 leader 的 proposal 时，根据接收的先后顺序处理 proposal。

# 3. 当 leader 收到 follower 针对某个 proposal 过半的 ack 后，则发起事务提交，重新发起一个 commit 的 proposal。

# 4. follower 收到 commit 的 proposal 后，记录事务提交，并把数据更新到内存数据库。

# 5. 当 写 成功后，反馈给 client。
```

#### 服务节点初始化同步

```shell
# 在集群运行过程当中，如果有一个 follower 节点宕机，集群正常运行，当 leader 收到新的客户端请求时，无法同步给宕机的节点，造成了数据不一致的问题。

# zk 为了解决这个问题，在节点启动时，第一件事就是找当前的 Leader，比对数据是否一致。

# 不一致，则开始同步，同步完成之后再进行对外提供服务。

# zk 利用 ZXID 事务ID 来进行数据的比对。
```

##### ZXID

```shell
# ZXID 是一个长度 64 位的数字。

# 其中低 32 位是按照数字递增，任何数据的变更都会导致其简单加1。

# 高 32 位是 leader 周期编号，每当选举出一个新的 leader 时，新的 leader 就从本地事务日志中取出 ZXID，然后解析出高 32 为的周期编号，进行加1，再将低 32 位的全部设置成0。

# 这样就保证了每次新选举 leader 后，ZXID 的唯一性且递增。
```

### 四字运维命令

```shell
# zk 响应少量命令，每个命令由四个字母组成。

# 可通过 telnet 或 nc 向 zk 发出命令。

# 这些命令默认是关闭的，需要配置 4lw.commands.whitelist 来打开。
	# 打开指定命令:
		# 4lw.commands.whitelist=stat, ruok, conf, isro
		
	# 打开全部:
		# 4lw.commands.whitelist=*
```

![UTOOLS1584943667654.png](http://yanxuan.nosdn.127.net/8b69136f656e74c1a78cd5fd60fbdb95.png)

![UTOOLS1584943759830.png](http://yanxuan.nosdn.127.net/1b146b7f2965af994e3851074fa4b381.png)

```shell
# 如果没有 nc 命令的，安装
yum install -y nc

# 查看服务器及客户端连接状态
echo stat|nc 127.0.0.1 2181
```

![UTOOLS1584943843683.png](http://yanxuan.nosdn.127.net/944a76bea6802b4fc663bc5fba24b839.png)

## zk 使用场景

### 分布式集群管理

#### 分布式集群管理的需求

```shell
# 主动查看线上服务节点

# 查看服务节点资源使用情况

# 服务离线通知

# 服务资源（CPU、内存、硬盘）超出阈值通知
```

#### 架构设计

![图片](https://uploader.shimo.im/f/5cdojbpemNQGhLaK.png!thumbnail)

```shell
# 利用 zk 的临时节点，watch 监听解决上述需求。
```

#### 节点结构

```shell
swordsman-manager # 根节点
	server00001:<json> # 服务节点1，临时序号节点
	server00002:<json> # 服务节点2，临时序号节点
	server....n:<json> # 服务节点n，临时序号节点
```

#### 核心代码

```java
/**
 * 数据监控
 * 
 * @Author DuChao
 * @Date 2020/3/24 2:42 下午
 */
public class Agent {

    private static Agent instance = new Agent();
    private String server = "127.0.0.1:2181";
    private ZkClient zkClient;
    private static final String rootPath = "/swordsman-manger";
    private static final String servicePath = rootPath + "/service";
    private String nodePath; // /swordsman-manger/service0000001 当前节点路径
    private Thread stateThread;

    public static Agent getInstance() { return instance; }

    private Agent() {}

    // javaagent 数据监控
    public static void premain(String args, Instrumentation instrumentation) {
        Agent.getInstance().init();
    }

    // 初始化工作
    public void init() {
        zkClient = new ZkClient(server, 5000, 10000);
        System.out.println("zk 连接成功! " + server);

        // 创建根节点
        buildRoot();

        // 创建临时节点
        createServerNode();

        // 启动更新的线程
        stateThread = new Thread(() -> {
            while (true) {
                // 更新节点数据
                updateServerNode();
                try {
                    TimeUnit.SECONDS.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "zk-state-Thead");
        stateThread.setDaemon(true);
        stateThread.start();

    }

    private void updateServerNode() {
        zkClient.writeData(nodePath, getOsInfo());
    }

    private void createServerNode() {
        nodePath = zkClient.createEphemeralSequential(servicePath, getOsInfo());
        System.out.println("创建节点: " + nodePath);
    }

    // 获取系统信息
    private Object getOsInfo() {
        OsBean bean = new OsBean();
        bean.setLastUpdateTime(DateUtil.now());
        bean.setIp(getLocalIp());
        bean.setCpu(CPUMonitorCalc.getInstance().getProcessCpu());

        MemoryUsage memoryUsage = ManagementFactory.getMemoryMXBean().getHeapMemoryUsage();
        bean.setUsableMemorySize(memoryUsage.getMax() / 1024 / 1024);
        bean.setUsedMemorySize(memoryUsage.getUsed() / 1024 / 1024);
        bean.setPid(ManagementFactory.getRuntimeMXBean().getName());

        return JSONUtil.toJsonStr(bean);
    }

    // 获取本地 Ip
    public static String getLocalIp() {
        InetAddress addr = null;
        try {
            addr = InetAddress.getLocalHost();
        } catch (UnknownHostException e) {
            throw new RuntimeException(e);
        }
        return addr.getHostAddress();
    }


    private void buildRoot() {
        // 如果根节点不存在
        if (!zkClient.exists(rootPath)) {
            zkClient.createPersistent(rootPath);
        }
    }

}
```

```java
@Controller
public class ZkController implements InitializingBean {

    @Value("${zk:127.0.0.1:2181}")
    private String server;
    private ZkClient zkClient;
    private static final String rootPath = "/swordsman-manger";
    Map<String, OsBean> map = new HashMap<>();

    @RequestMapping("/list")
    public String list(Model model) {
        model.addAttribute("items", getCurrentOsBeans());
        return "list";
    }

    // 获取当前节点列表信息
    private Object getCurrentOsBeans() {
        List<OsBean> items = zkClient.getChildren(rootPath)
                .stream()
                .map(v -> rootPath + "/" + v)
                .map(v -> convert(zkClient.readData(v)))
                .collect(Collectors.toList());
        return items;
    }

    private OsBean convert(String json) {
        return JSONUtil.toBean(json, OsBean.class);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        zkClient = new ZkClient(server, 5000, 10000);
        initSubscribeListener();
    }

    // 初始化订阅
    private void initSubscribeListener() {
        // 解除所有监听
        zkClient.unsubscribeAll();
        // 获取所有子节点
        zkClient.getChildren(rootPath)
                .stream()
                .map(v -> rootPath + "/" + v) // 得到子节点的完整路径
                .forEach(v -> zkClient.subscribeDataChanges(v, new DataChanges())); // 监听数据变更
        // 监听子节点的变更: 增加、删除
        zkClient.subscribeChildChanges(rootPath, (parentPath, currentChilds) -> initSubscribeListener());
    }

    // 子节点数据变化
    private class DataChanges implements IZkDataListener {

        @Override
        public void handleDataChange(String dataPath, Object data) throws Exception {
            OsBean bean = convert((String) data);
            map.put(dataPath, bean);
            doFilter(bean);
        }

        @Override
        public void handleDataDeleted(String dataPath) throws Exception {
            if (map.containsKey(dataPath)) {
                OsBean bean = map.get(dataPath);
                System.err.println("服务已下线:" + bean);
                map.remove(dataPath);
            }
        }
    }

    // 警告过滤
    private void doFilter(OsBean bean) {
        // cpu 超过10% 报警
        if (bean.getCpu() > 10) {
            System.err.println("CPU 报警..." + bean.getCpu());
        }
    }

}
```

#### 实现效果

![UTOOLS1585038068290.png](http://yanxuan.nosdn.127.net/34a835545ea0e4956e0ad1162abdfcad.png)

### 分布式注册中心

```shell
# 在单体服务中，通常是多个服务端调用一个客户端，只要在客户端写死唯一服务地址即可。

# 当升级到分布式系统后，服务端节点变多，写死配置就不可靠了。

# 所以此时需要一个中间服务，专门用于帮助客户端发现服务节点，即服务发现。
```

![图片](https://uploader.shimo.im/f/l1l3j9Jxt4AsAxTT.png!thumbnail)

```shell

```

