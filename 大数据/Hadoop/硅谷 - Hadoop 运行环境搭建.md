# 硅谷 - Hadoop 运行环境搭建

## 安装 JDK

```shell
# 参考 AgeFades-Note/运维/Linux 基础环境搭建
```

## 安装 Hadoop

```shell
# Hadoop 下载地址
https://archive.apache.org/dist/hadoop/common/hadoop-2.7.2/

# 上传 hadoop-2.7.2.tar.gz 至服务器并解压
/opt/module/hadoop-2.7.2

# 添加环境变量
vim /etc/profile

# 末尾添加
export HADOOP_HOME=/opt/module/hadoop-2.7.2
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin

# 保存退出，并让它生效
source /etc/profile

# 检测
hadoop version
```

## Hadoop 目录结构

![UTOOLS1574149063409.png](https://i.loli.net/2019/11/19/8BDnKFeGpdwQPcI.png)

```shell
# bin : 存放对 Hadoop 相关服务<HDFS，YARN> 进行操作的脚本

# etc : Hadoop 的配置文件目录，存放Hadoop 的配置文件

# lib : 存放 Hadoop 的本地库<对数据进行压缩解压缩功能>

# sbin : 存放启动或停止 Hadoop 相关服务的脚本

# share : 存放 Hadoop 的依赖 Jar 包、文档、官方案例
```

## Hadoop 运行模式

```shell
# Hadoop 运行模式包括:
	# 本地模式
	# 伪分布式模式
	# 完全分布式模式
```

### 本地运行模式

#### 官方 Grep 案例

```shell
# 在 hadoop-2.7.2 下面创建一个 input 文件夹
mkdir input

# 将 hadoop 的 xml 配置文件复制到 input
cp etc/hadoop/*.xml input

# 执行 share 目录下的 MapReduce 程序
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar grep input output 'dfs[a-z]+'

# 查看输出结果
cat output/*
```

### 官方 WordCount 案例

```shell
# 在 hadoop-2.7.2 下面创建一个 wcinput 文件夹
mkdir wcinput

# 在 wcinput 文件下创建一个 wc.input 文件
cd wcinput
touch wc.input

# 编辑 wc.input
vim wc.input

# 输入以下内容
hadoop yarn
hadoop mapreduce
agefades
agefades

# 回到 hadoop-2.7.2
cd ../

# 执行程序
hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount wcinput wcoutput

# 查看结果
cat wcoutput/part-r-00000
```

### 伪分布式运行模式

#### 启动 HDFS 并运行 MapReduce 程序

##### 分析

```shell
# 配置集群

# 启动、测试集群增、删、查

# 执行 WordCount 案例
```

##### 执行步骤

```shell
# 配置集群
vim etc/hadoop/core-site.xml 

# 在 <configuration> 标签里添加:
```

```xml
<!-- 指定HDFS中NameNode的地址 -->
<property>
<name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
</property>

<!-- 指定Hadoop运行时产生文件的存储目录 -->
<property>
  <name>hadoop.tmp.dir</name>
  <value>/opt/module/hadoop-2.7.2/data/tmp</value>
</property>
```

```shell
# 配置 hadoop-env.sh
echo ${JAVA_HOME}

# 将 ${JAVA_HOME} 路径拷贝，编辑文件:
vim etc/hadoop/hadoop-env.sh

# 修改配置
export JAVA_HOME=/platform/jdk1.8.0_211 # 这里填自己的 ${JAVA_HOME} 路径

# 配置 hdfs-site.xml
vim etc/hadoop/hdfs-site.xml

# 在 <configuration> 标签里添加:
```

```xml
<!-- 指定HDFS副本的数量 -->
<property>
	<name>dfs.replication</name>
	<value>1</value>
</property>
```

```shell
# 启动集群
	# 格式化 NameNode <第一次启动时格式化，以后不用总格式化>
bin/hdfs namenode -format

	# 启动 NameNode
sbin/hadoop-daemon.sh start namenode

	# 启动 DataNode
sbin/hadoop-daemon.sh start datanode

# 查看集群
	# 查看是否启动成功
jps

	# web 端查看 HDFS 文件系统
['ip']:50070/dfshealth.html#tab-overview

	# 查看产生的 log
cd logs
cat hadoop-['当前用户']-datanode-['系统主机名'].log

# 为什么不能一直格式化 NameNode, 格式化 NameNode 要注意什么？
cd ../data/tmp/dfs/name/current/
cat VERSION	# 查看集群Id

	# 格式化 NameNode，会产生新的集群id，导致 NameNode 和 DataNode 的集群Id 不一致，集群找不到以往数据。
	# 所以，格式化 NameNode 时，一定要先删除 data 数据和 log 日志，然后再格式化 NameNode
```

##### 操作集群

```shell
# 在 HDFS 文件系统上创建一个 input 文件夹
bin/hdfs dfs -mkdir -p /user/agefades/input

# 将测试文件内容上传到文件系统上
bin/hdfs dfs -put wcinput/wc.input /user/agefades/input/
 
# 查看上传的文件是否正确
bin/hdfs dfs -ls /user/agefades/input/
bin/hdfs dfs -cat /user/agefades/input/wc.input

# 运行 MapReduce 程序
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.2.jar wordcount /user/agefades/input/ /user/agefades/output

# 查看输出结果
bin/hdfs dfs -cat /user/agefades/output/*

# 将测试文件内容下载到本地
hdfs dfs -get /user/agefades/output/part-r-00000 ./wcoutput/

# 查看运行结果
cat wcoutput/part-r-00000

# 删除输出结果
hdfs dfs -rm -r /user/agefades/output
```

#### 启动 YARN 并运行 MapReduce 程序

##### 分析

```shell
# 配置集群在 YARN 上运行 MR

# 启动、测试集群增、删、查
```

