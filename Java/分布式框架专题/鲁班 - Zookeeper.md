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
# ACL 全程为
```

