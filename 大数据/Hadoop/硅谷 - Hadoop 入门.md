# 硅谷 - Hadoop 入门

## Hadoop 介绍

### Hadoop 是什么？

```shell
# Hadoop 是一个由 Apache 开发的分布式系统基础架构

# 主要解决:
	# 海量数据的存储和海量数据的分析计算问题
	
# 广义上来说，Hadoop 通常是指一个更广泛的概念 -> Hadoop 生态圈
```

### Hadoop 发展历史

```shell
# Lucene ，一款由Java 编写的框架，实现与 Google 类似的全文搜索功能，提供了全文检索引擎的架构，包括完整的查询引擎和索引引擎。

# 2001年，Lucene 加入 Apache

# 对于海量数据场景，Lucene 面对与 Google 同样的困难，存储数据困难，检索数据慢

# 学习与模仿 Google 解决这些问题的办法 : 微型版 Nutch

# Google 是 Hadoop 的思想之源 <三篇论文>
	# GFS -> HDFS
	# Map-Reduce -> MR
	# BigTable -> HBase
	
# 2003 - 2004 年，Google 公开部分 GFS 和 MapReduce 思想的细节，以此为基础 Doug Cutting 等人用了 2年业余时间实现了 DFS 和 MapReduce 机制，使 Nutch 性能飙升

# 2005 年，Hadoop 作为 Lucene 的子项目，Nutch 的一部分正式引入 Apache 基金会

# 2006 年，MapReduce 和 NDFS 分别纳入到 Hadoop 项目，Hadoop 正式诞生，标志大数据时代来临
```

### Hadoop 三大发行版本

```shell
# Apache 版本

# Cloudera 版本

# Hortonworks 版本
```

### Hadoop 的优势

```shell
# 高可靠性:
	# Hadoop 底层维护多个数据副本
	
# 高扩展性:
	# 在集群间分配任务数据，可方便的扩展数以千计的节点
	
# 高效性:
	# 在 MapReduce 的思想下，Hadoop 是并行工作的，加速任务处理
	
# 高容错性:
	# 能够自动将失败的任务重新分配
```

### Hadoop 组成

```shell
# Hadoop1.x
	# Common : 辅助工具
	# HDFS : 数据存储
	# MapReduce : 计算 + 资源调度
	
# Hadoop2.x
	# Common : 辅助工具
	# HDFS : 数据存储
	# Yarn : 资源调度
	# MapReduce : 计算
	
# 在Hadoop1.x 时代，MapReduce 同时处理业务逻辑运算和资源的调度，耦合性较大，在 Hadoop2.x 时代，增加了 Yarn。Yarn 只负责资源调度，MapReduce 只负责运算。
```

#### HDFS 架构概述

```shell
# Hadoop Distributed File System <HDFS 全名>
	# NameNode:
		# 存储文件的元数据，如文件名，文件目录结构，文件属性(生成时间、副本数、文件权限)，以及每个文件的块列表和块所在的 DataNode 等。
		
	# DateNode:
		# 在本地文件系统存储文件块数据，以及数据块的校验和。
		
	# Secondary NameNode:
		# 用来监控 HDFS 状态的辅助后台程序，每隔一段时间获取 HDFS 元数据的快照。
```

#### YARN 架构概述

```shell
# ResourceManager:
	# 处理客户端请求
	# 监控 NodeManager 
	# 启动或监控 ApplicationMaster
	# 资源的分配与调度
	
# NodeManager:
	# 管理单个节点上的资源
	# 处理来自 ResourceManager 的命令
	# 处理来自 ApplicationMaster 的命令
	
# ApplicationMaster:
	# 负责数据的切分
	# 为应用程序申请资源并分配给内部的任务
	# 任务的监控与容错
	
# Container:
	# Container 是 YARN 中的资源抽象，它封装了某个节点上的多维度资源，如内存、CPU、网络等
```

#### MapReduce 架构概述

```shell
# 计算过程的两个阶段 -> Map And Reduce
	# Map 阶段并行处理输入数据
	# Reduce 阶段对 Map 结果进行汇总

```

## 大数据技术生态体系

### 生态体系图

![UTOOLS1573110851809.png](https://i.loli.net/2019/11/07/QBuDbF571IEHo6R.png)

### 名词解释

```shell
# Sqoop:
	# 一款开源工具，主要用于在 Hadoop、Hive 与传统的数据库<Mysql> 间进行数据的传递，可以将一个关系型数据库中的数据导进到 Hadoop 的 HDFS 中，也可以将 HDFS 的数据导进到关系型数据库中。
	
# Flume:
	# Flume 是 Cloudera 提供的一个高可用、高可靠、分布式的海量日志采集、聚合和传输的系统，Flume 支持在日志系统中定制各类数据发送方，用于收集数据。
	# 同时，Flume 提供对数据进行简单处理，并写到各种数据接收方<可定制> 的能力。
	
# Kafka:
	# Kafka 是一种高吞吐量的分布式发布订阅消息系统，有如下特性:
		# 通过 0(1) 的磁盘数据结构提供消息的持久化，这种结构对于即使数以 TB 的消息存储也能够保持长时间的稳定性能。
		# 高吞吐量: 即便是非常普通的硬件 Kafka 也可以支持每秒数百万的消息。
		# 支持通过 Kafka 服务器和消费机集群来区分消息。
		# 支持 Hadoop 并行数据加载。
		
# Storm:
	# Storm 用于"连续计算"，对数据流做连续查询，在计算时就将结果以流的形式输出给用户。
	
# Spark:
	# Spark 是当前最流行的开源大数据内存计算框架。可以基于 Hadoop 上存储的大数据进行计算。
	
# Oozie:
	# Oozie 是一个管理 Hadoop 作业<job> 的工作流程调度管理系统。
	
# HBase:
	# HBase 是一个分布式、面向列的开源数据库。
	# 不同于一般的关系型数据库，它是一个适合于非结构化数据存储的数据库。
	
# Hive:
	# Hive 是基于 Hadoop 的一个数据仓库工具，可以将结构化的数据文件映射为一张数据库表，并提供简单的 SQL 查询功能，可以将 SQL 语句转换为 MapReduce 任务进行运行。
	
# R 语言:
	# R 是用于统计分析、绘图的语言和操作环境。
	
# Mahout:
	# Apache Mahout 是个可扩展你的机器学习和数据挖掘库。
	
# Zookeeper:
	# Zookeeper 是 Google 的 Chubby 一个开源实现，是一个针对大型分布式系统的可靠协调系统。
	# 提供的功能包括:
		# 配置维护
		# 名字服务
		# 分布式同步
		# 组服务等
```

### 推荐系统框架图

![UTOOLS1573113614307.png](https://i.loli.net/2019/11/07/2c781DUFqWIxhRJ.png)

