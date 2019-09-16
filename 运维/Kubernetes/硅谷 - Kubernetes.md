# 硅谷 - Kubernetes

## 组件介绍

### 发展经历

```shell
# IAAS 基础设施即服务 -> 典型代表<阿里云>
# PAAS 平台即服务 -> 典型代表<新浪云>
# SAAS 软件即服务 -> 典型代表<Office>

# 其中 PAAS 为免运维，为简化代码运行基础环境的搭建，诞生 Docker
# 而 Docker 容器之间的互联、资源分配、扩容等等问题也随之而来

# Apache MESOS , 最古老的分布式资源管理框架，Twitter 最早采用并大规模应用，19年5月全部换为 K8S
# Docker SWARM , 已成为 Docker 的一个内部组件，非常轻量，但是目前功能不如 K8S 远矣
	# 19年7月 阿里云宣布 Docker Swarm 集群框架剔除
# Kubernetes , 目前主流，Google 10年前就已经容器化基础架构，
	# 是其用 Go 语言对其内部基础设施框架的 Borg 进行翻写出来的框架
	
# K8S 特点
	# 轻量级 Go 语言解释性语言，效率很高<可以看成进化版的 C 语言>，消耗资源小
	# 开源
	# 弹性伸缩
	# 负载均衡 : 最新版本中采用 IPVS 作负载
```

### 知识图谱

```shell
# 请看附件 Xmind
```

### 组件说明

```shell
# 先看 Borg 系统架构图
```

![UTOOLS1568254666830.png](https://i.loli.net/2019/09/12/hVAb49MdJcKrs16.png)

```shell
# BorgMaster
	# 负责请求分发，集群数需为奇节点<投票机制>

# Borglet 
	# 真正的工作节点

# scheduler
	# 真正的调度器
	
# 流程
	# borgcfg | command-line tools | web browsers <请求来源> 
# -> BorgMaster 集群
# -> 调用 scheduler 计算 KV 值存入 Paxos 键值库
# -> Borglet 监听 Paxos，取出计算后 K 值匹配的请求并进行消费
	
```

```shell
# 再来看 K8S 的架构图
```

![UTOOLS1568255144081.png](https://i.loli.net/2019/09/12/bjfRwp2Seo9WBiH.png)

```shell
# C/S 结构、主要分为 Master 服务器、Node 节点

# scheduler 
	# 还是任务调度器，接收请求，通过算法分配到合适的节点
	# 不同的是，scheduler 在 K8S 中将任务交给 Api Servce，再由 Api Server 写入 etcd
	
# replication controller<简写 rc>
	# 控制器，负责维护副本的数目<例 : 期望10个 MySQL 容器节点Pod，此时即 rc 控制>
	
# Api Server
	# 所有组件请求入口
	# 为缓解 Api Server 压力，每个组件都是有自己的缓存的

# etcd
	# 类似于 Borg 系统中的 Paxos
	# Chrome 用 Go 语言编写的分布式 KV 库
	# 整个 K8S 的持久化仓库
	# 无需其他组件，自己就可以做集群化
	# V2 版本中，将所有数据写入 Memory<内存>，K8S v1.11 后已弃用
	# V3 版本中，将所有数据写入 Database<本地磁盘>，建议使用
	
# 再到 Node 架构
# Kubelet
	# CRI <Container Runtime Interface>
	# 操作 Docker 创建对应容器，维持 Pod 生命周期

# Kube proxy
	# 完整负载均衡的操作
	# 默认操作 firewall<防火墙>，实现 Pod 的映射
	# 最新版本中支持 IPVS

# Pod
	# container , 默认为 Docker 容器
	
# 其他重要插件说明
	# CoreDNS : 为集群中的 SVC<Service> 创建一个域名Ip 对应的关系解析
	
	# Dashboadr # B/S架构，给 K8S 提供的一个网页访问体系
	
	# Ingress Contpoller : 官方只能实现 4 层代理，Ingress 可以实现七层代理
	
	# Fedetation : 提供一个可以垮集群中心多 K8S 的统一管理功能
	
	# Prometheus : 提供 K8S 集群的监控能力
	
	# ELK : 提供 K8S 集群日志统一分析接入平台
	
```

```shell
# 来看看 etcd 的架构图
```

![UTOOLS1568255727386.png](https://i.loli.net/2019/09/12/M3SNidD9fgR1z6T.png)

```shell
# HTTP Server
	# 意味着 etcd 依然是一个采用 http 通信的 C/S 服务<K8S 也是>
	# HTTP 较之 TCP，携带请求包会比较大，通信速度慢，但是支持 GET|POST|... 甚至认证协议，功能更强大
	
# Raft
	# 读写信息
	# WAL 读写日志
	# Entry | Snapshot 定时做一些完整的、临时的备份

# Store
	# Raft 还会实时的将数据存储到本地化 Store
```

## 基础概念

### Pod 概念

```shell
# K8S 中Node服务的最小节点
# 每个 Pod 中都有一个 Pause 容器<管理Pod 网络栈、存储卷>
# 每个 Pod 中除了 Pause 外，还有 >=1 的容器服务
# Pod 中各个容器可以通过 Localhost 互访、并且端口不能冲突
```

#### 自主式 Pod

```shell
# 不会有任何控制器对其管理，其生命周期、期望副本数 等都是没有被控制的。
```

#### 控制器管理的 Pod

```shell
# Pod 控制器类型
# ReplicationController
	# 该控制器用来确保应用的副本数始终保持在用户定义的副本数
	# 容器异常退出 -> 自动创建新的 Pod 替代
	# 容器异常多出 -> 自动回收
	# 新版本 K8S 中，建议使用 ReplicaSet 来取代 rc<简写> 
	
# ReplicaSet
	# 与 rc 没有本质不同，功能的增加
	# 支持集合式的 selector<Pod 的标签，例如 : App - Apache Or Version - 1.0>
	# 虽然 ReplicaSet 可以独立使用，但一般还是建议使用 Deployment 来自动管理 ReplicaSet
		# 这样就无需担心跟其他机制不兼容的问题(例如: rs 不支持 rolling-update,但 Deployment 支持)
		
# HPA(Horizontal Pod AutoScale)
	# 仅适用于 Deployment 和 ReplicaSet
	# 在 V1 版本中仅支持根据 Pod 的 CUP 利用率伸缩<扩容 | 缩容>
	# 在 vlalpha 版本中，支持根据内存和用户自定义的 metric 伸缩
	
# StatefulSet
	# 是为了解决有状态服务的问题<对应 Deployments 和 rs 是为无状态服务而设计>
	# 应用场景包括 :
		# 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 实现
		# 稳定的网络标志，即 Pod 重新调度后其 PodName 和 HostName 不变，
			# 基于 Headless Service 实现<即没有 Cluster Ip 的 Service>
		# 有序部署、有序扩展。即 Pod 是有顺序的，在部署或者扩展的时候，依据顺序依次进行
			# 在下一个Pod 运行之前，所有其之前的 Pod 都应该是 Running Or Ready 状态
			# 基于 init containers 实现
		# 有序收缩、有序删除。<即从 N-1 到 0 >
		
# DaemonSet
	# 确保全部(或者一些) Node 上运行一个 Pod 的副本。
	# 当有Node 加入集群时，也会为他们新增一个 Pod。
	# 当有Node 从集群移除时，这些 Pod 也会被回收
	# 删除 DaemonSet，将会删除它创建的所有 Pod
	# 典型用法 :
		# 运行集群存储 daemon，例如在每个 Node 上运行 glusterd、ceph
		# 在每个 Node 上运行日志收集 daemon，例如 fluentd、logstash
		# 在每个 Node 上运行监控 daemon，例如 Prometheus Node Exporter
		
# Job And Cron Job
	# Job 负责批处理任务，即仅执行一次的任务，它保证批处理任务的一个或多个 Pod 成功结束
	# Cron Job 即 定时任务
		# 在给定时间内只运行一次
		# 周期性的给定时间点运行
```

```shell
# 服务发现
# 流程
	# Client<客户端> 
	# -> Service<提供统一的 Ip、Port 等> 轮询算法
	# -> rs Or 其他控制器定义下的一组Pod<例 : 相同的标签(App - Apache...)>
```

### 网络通讯方式

```shell
# K8S 的网络模型假定了所有 Pod 都在一个可以直接连通的扁平的网络空间中。
	# 这在 GCE<Google Compute Engine> 里面是现成的网络模型，K8S 假定这个网络已经存在
	# 而在私有云里搭建 K8S 集群，就不能假定这个网络已经存在了
	# 我们需要自己实现这个网络假设，将不同节点上的 Docker 容器之间的互相访问打通，再运行 K8S
	
# 同一个 Pod 内的多个容器之间
	# 是通过 Pod 内部的 Pause 容器，共用同一个网络栈<Linux 协议栈>
	# 容器之间通过 localhost 互访
	
# 各 Pod 之间的通讯
	# Overlay Network
	# 在同一台机器上，由 Docker0 网桥直接转发请求至 Pod2，不需要经过 Flannel
	# 不在同一台机器上时，Pod 的地址是与Docker0 在同一个网段的，
		# 但Docker0 与宿主机网卡是两个完全不同的Ip 网段，并且不同 Node 之间只能通过宿主机的 Ip 通信
		# 将 Pod 的 Ip 和所在 Node 的Ip 关联起来，放可实现 Pod 的互访
		
# Pod 与 Service 之间的通信
	# 各节点的 Iptables 规则<DNAT 路由规则，详情可参见同目录下 Iptables 规则笔记>
	# 最新版本 K8S 已经采用 LVS
	
# 外网访问 Pod
	# 通过 Service
	
# Pod 到外网
	# Pod 向外网发送请求，查找路由表，转发数据包到宿主机的网卡，宿主网卡完成路由选择后
		# Iptables 执行 Masquerade，把源 Ip 更改为宿主网卡的 Ip,然后向外网发送请求
	
# Flannel 是 CoreOS 团队针对 K8S 设计的一个网络规划服务
	# 让集群中的不同节点主机创建的 Dokcer 容器都具有全集群唯一的虚拟 Ip 地址
	# 还能再这些 Ip 地址之间建立一个虚拟网络<Overlay NetWork>
	# 通过该虚拟网络，将数据包原封不动地传递到目标容器内
	# etcd 存储管理 Flannel 可分配的 IP 地址段资源
	# 监控 etcd 中每个 Pod 的实际地址，并在内存中建立维护 Pod 节点路由表

# K8S 中的三层网络架构图，如下	
	# 其中，只有节点网络为真实网卡
	# Service 与 Pod 的网络都是虚拟网卡
```

![UTOOLS1568273686997.png](https://i.loli.net/2019/09/12/GcVWa61MeSDJQyP.png)

## 集群安装

```shell
# 学习环境搭建架构图，如下 : 
```

![UTOOLS1568274731980.png](https://i.loli.net/2019/09/12/ZAwcqW9EPL8JChd.png)

```shell
# 此处教程为虚拟机操作，并用 koolshare 搭建软路由进行科学上网
	# 升级系统内核为 4.4，重启虚拟机。
	
# 因本人使用为公司服务器，上面跑各种服务，无法拿来一顿操作，故 略。
# 找到一篇参考文章
https://www.jianshu.com/p/e2412cf0d68f
```

## 常用命令

```shell
# 以下命令摘抄自 https://www.jianshu.com/p/d7784557ff61
# K8S 命令的组成
	# kubectl 动作 资源 [选项]
	
# kubectl 命令的基础动作
	# get : 展示一个或多个资源
	# create : 通过资源配置文件名或者键盘输入创建资源
	# expose : 选择一个 rs、service、deployment、Pod，并且暴露为新的 K8S 服务
	# run : 在集群上运行指定镜像
	# set : 在对象上设置指定属性
	# explain : 资源的文档
	# edit : 编辑服务器上的资源
	# delete : 通过资源控制的文件名，键盘输入，资源名，或者选择器标签等删除资源
	
# kubectl 提供的高级语法
	# 部署类
		# rollout : 管理 deployment 的部署
		# roulling-update : 实现滚动升级，并最终输出 rs
		# scale : 为 deployment、replicaSet、rs、job 设置新的大小
		# autoscale : 自动伸缩 deployment、replicaSet ...
	
	# 集群管理类
		# certificate : 修改认证资源
		# cluster-info : 显示集群信息
		# top : 显示(Cpu | Memory | Storage) 资源的使用
		# cordon : 标记节点为 unschedulable
		# uncordon : 标记节点为 schedulabel
		# drain : 排除节点，准备维修
		# taint : 更新一个或多个节点上的污点
		
	# 故障定位和排除类的命令
		# describe : 显示指定资源或者资源组的详情
		# logs : 打印某个 Pod 中容器的日志
		# attach : 附加在一个运行的容器上执行，使用该命令注意不要关闭容器并退出
		# exec : 在一个容器中执行命令，不影响现在运行的容器中的功能
		# port-forward : 转发一个或多个端口到 Pod 中
		# proxy : 运行 proxy 以实现到 K8S Api Server 的功能转发
		# cp : 与容器之间进行文件拷贝
		
	# 其他更高级的命令
		# apply、patch、replace、convert ...
		
	# 设置命令
		# label、annotate、completion ...
		
	# 其他系统级命令
		# api-versions : 以 group/version 的形式打印服务器上支持的 API 版本
		# config : 修改 kubeconfig 文件
		# help : 帮助命令
		# version : 打印客户端和服务器的版本号
		
# get
	# get 是 kubectl 中最基础的命令
	# 格式 : get 资源 [选项]
	# 几个常用的 get 命令组合
		# 默认会隐藏一些资源信息，如运行情况等，要显示的话在命令最后加 --show-all
		# kubectl get pods : 显示所有的Pod 信息，格式如 Linux 中的 ps 命令<精简>
		# kubectl get pods -o wide : 全面显示Pod 信息
		# kubectl get replicationSet [rs 名] : 查看单个指定 rs 名称的信息
		# kubectl get -o json pod [pod 名] : 使用 Json 格式展示指定的 Pod 信息
		# kubectl get -f pod.yaml -o json : Json 格式展示 yaml形式的 Pod 信息
		# kubectl get rs,service : 同时输出所有的 rs 和 service 资源实例列表
		# kubectl get rs/[rs 名] service/[service 名] pods/[pod 名] : 获取具体实例信息
	
	# get 中的重要选项
		# --all-namespaces=false : 跨命名空间查询对象
		# -f Or --filename=[] : 指定配置文件名
		# -o Or --output='' : 指定输出格式
		# -a : 输出所有的信息
```

## 资源清单

### K8S 中的资源

```shell
# 集群资源分类
	# 名称空间级别
		# 通过 kubeadmin 安装的 K8S，默认所有组件都会放在 kube-system 下
		# 所以 kubectl get pod<等于 kubectl get pod -n default> 是获取不到系统的Pod 信息
		# 这就是资源的名称空间级别区分
	# 集群级别
		# 全局资源，不管在什么地方定义，其他地方都能看到<定义时都没有指定名称空间>
	# 元数据型
		# 类似 HPA，通过指标去控制资源的平滑伸缩
		
# 什么是资源？
	# K8S 中所有的内容都抽象为资源，资源实例化后，叫做对象
	
# K8S 中存在哪些资源？
	# 名称空间级别
		# 工作负载型资源(workload)
			# Pod、ReplicaSet、Deployment、StatefulSet、DaemonSet、
			# Job、CronJob<rc v1.11版本后被废弃>
		# 服务发现及负载均衡型资源(ServiceDiscovery LoadBalance)
			# Service、Ingress、...
		# 配置与存储型资源
			# Volume(存储卷)、CSI(容器存储接口、可以扩展各种各样的第三方存储卷)
		# 特殊类型的存储卷
			# ConfigMap(当配置中心来使用的资源类型)、Secret(保存敏感数据)、
			# DownwardAPI(把外部环境中的信息输出给容器)
		
	# 集群级资源
		# Namespace、Node、Role、ClusterRole、RoleBinding、ClusterRoleBinding
		
	# 元数据型资源
		# HPA、PodTemplate、LimitRange

```

### 资源清单

```shell
# 资源清单含义
	# 在 K8S 中，一般使用 yaml 格式的文件来创建符合预期的 Pod，这样的 yaml 文件称为资源清单
```

### 常用字段解释说明

```shell
# 必须存在的属性
```

| 参数名                  | 字段类型 | 说明                                                         |
| ----------------------- | -------- | ------------------------------------------------------------ |
| version                 | String   | K8S API 的版本，目前基本上是v1<br />可以用 kubectl api-version 命令查询 |
| kind                    | String   | yaml 文件定义的资源类型和角色，例 : Pod                      |
| metadate                | Object   | 元数据对象，固定值就写 metadata                              |
| metadata.name           | String   | 元数据对象的名字，比如命名 Pod 的名字                        |
| metadata.namespace      | String   | 元数据对象和命名空间，自定义                                 |
| Spec                    | Object   | 详细定义对象，固定值就写 Spect                               |
| spec.containers[]       | list     | Spec 对象的容器列表定义，是个列表                            |
| spec.containers[].name  | String   | 定义容器名字                                                 |
| spec.containers[].image | String   | 定义要用到的镜像名称                                         |

```shell
# 主要对象
	# 以下全是 spec.containers[] 的子属性，故略
	# 举例 spec.containers[].imagePullPolicy
```

| 参数名                   | 字段类型 | 说明                                                         |
| ------------------------ | -------- | ------------------------------------------------------------ |
| imagePullPolicy          | String   | 定义镜像拉取策略<br />Always : 每次都尝试重新拉取镜像<br />Never : 仅使用本地镜像<br />ifNotPresent : 没有本地镜像时，拉取线上镜像<br />默认是 Always |
| command[]                | list     | 指定容器启动命令<br />不指定则使用镜像打包时使用的启动命令   |
| args[]                   | list     | 指定容器的启动命令参数                                       |
| workingDir               | String   | 指定容器的工作目录                                           |
| volumeMounts[]           | list     | 指定容器内部的存储卷配置                                     |
| volumeMounts[].name      | String   | 指定可以被容器挂载的存储卷的名称                             |
| volumeMounts[].mountPath | String   | 指定可以被容器挂载的存储卷的路径                             |
| volumeMounts[].readOnly  | String   | 设置存储卷路径的读写模式，默认为 true                        |
| ports[]                  | list     | 指定容器需要用到的端口列表                                   |

| 参数名                | 字段类型 | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| ports[].name          | String   | 指定 端口名称                                                |
| ports[].containerPort | String   | 指端容器需要监听的端口号                                     |
| ports[].hostPort      | String   | 指定容器所在主机需要监听的端口号<br />默认跟上面 containerPort 相同<br />设置了hostPort 同一台主机无法启动该容器的相同副本 |
| ports[].protocol      | String   | 指定端口协议，支持 TCP 和 UDP，默认 TCP                      |
| env[]                 | list     | 指定容器运行前需设置的环境变量列表                           |
| env[].name            | String   | 指定环境变量名称                                             |
| env[].value           | String   | 指定环境变量值                                               |
| resources             | Object   | 指定资源限制和资源请求的值<br />开始就是设置容器的资源上限   |
| resources.limits      | Object   | 指定设置容器运行时资源的运行上限                             |

| 参数名                    | 字段类型 | 说明                                                         |
| ------------------------- | -------- | ------------------------------------------------------------ |
| resources.limits.cpu      | String   | 指定CPU 的限制，单位为 core 数<br />用于 docker run --cpu-shares 参数 |
| resources.limits.memory   | String   | 指定MEM 内存的限制，单位为 MIB、GIB                          |
| resources.requests        | Object   | 指定容器启动和调度时的限制设置                               |
| resources.requests.cpu    | String   | CPU 请求、单位为 core 数，容器启动时初始化可用数量           |
| resources.requests.memory | String   | 内存请求、单位为 MIB、GIB，容器启动时初始化可用数量          |

```shell
# 额外的参数项
```

| 参数名                | 字段类型 | 说明                                                         |
| --------------------- | -------- | ------------------------------------------------------------ |
| spec.restartPolicy    | String   | 定义Pod 的重启策略<br />Always : Pod 一旦终止运行，kubelet 服务都会重启它<br />OnFailure : Pod 以非零退出码终止时，kubelet 才会重启该容器<br />Never : Pod 终止后，kubelet 将退出码报告给 Master，不会重启该 Pod |
| spec.nodeSelector     | Object   | 定义Node 的Label 过滤标签，以 key:value 格式指定             |
| spec.imagePullSecrets | Object   | 定义pull镜像时使用secret名称，以 name:secretkey 格式指定     |
| spec.hostNetwork      | Boolean  | 定义是否使用主机网络模式，默认为 false<br />true 使用主机网络，而不是 docker0 网桥<br />true 时无法在同一台宿主机上启动第二个副本 |

### 容器生命周期

![UTOOLS1568613183500.png](https://i.loli.net/2019/09/16/pl37Pbci9sjJvxK.png)

```shell
# Pod 从创建到死亡的过程
	# 请求指令 
	# -> Api 接口 
	# -> etcd 记录指令、调度分发指令
	# -> 调度到 K8S 上 
	# -> 容器环境的初始化<Pause 创建>
	# -> 容器初始化<没有数量限制，线性初始化>
	# -> 执行容器运行前指令
	# -> 就绪检测
	# -> 主容器运行
	# -> 生存检测
	# -> 执行容器退出前指令
```

```shell
# Init 容器
# Pod 能具有多个容器，应用运行在Pod 里面，但是它也可能有一个或多个先于应用容器启动的 Init 容器
	# Init 容器与普通容器非常像，除了如下两点
		# Init 容器总是运行到成功完成为止
		# 每个 Init 容器都必须在下一个 Init 容器启动之前成功完成
		
	# 如果Pod 的 Init 容器失败，K8S 会不断重启该 Pod，直到成功为止
		# 除非 Pod 对应的 restartPolicy 为 Never
		
# Init 容器的作用
	# 因为 Init 容器具有与应用程序容器分离的单独镜像，所有它们的启动相关代码具有如下优势
		# 可以包含并运行实用工具，但出于安全考虑，不建议在应用程序容器镜像中包含这些实用工具的
		# 可以包含实用工具和定制化代码来安装，但是不能出现在应用程序镜像中。
			# 例如，创建镜像没必要 FROM 另一个镜像，只需安装中实用类似 sed、awk、python 等工具
		# 应用程序镜像可以分离出创建和部署的角色，而没必要联合它们构建一个单独的镜像
		# Init 容器使用 Linux NameSpace，所以相对应用程序容器来说，具有不同的文件系统视图。
			# 因此，它们能具有访问 Secret 的权限，而应用程序容器则不能
		# 必须在应用程序容器启动之前完成，而应用程序容器是并行运行的，所以 Init 容器提供一种简单的
			# 阻塞或延迟应用容器的启动的方法，知道满足一组先决条件
			
# 特殊说明
	# 在Pod 启动过程中，Init 容器会按顺序在网络和数据卷初始化之后启动，
		# 每个容器必须在下一个容器启动之前成功退出
		
	# 如果由于运行时或失败退出，将导致容器启动失败，它会根据 Pod 的 restartPolicy 策略重试
	
	# 在所有的 Init 容器没有成功之前，Pod 将不会变成 Ready 状态。
		# Init 容器的端口将不会在 Service 中进行聚集
		# Pod 处于 Pending 状态，但应该会将 Initializing 状态设置为 true
	
	# 如果 Pod 重启，所有 Init 容器必须重新执行
	
	# 对 Init 容器 spec 的修改被限制在容器 image 字段，修改其他字段都不会生效
		# 更改 Init 容器的 image 字段，等价于重启该 Pod
		
	# Init 容器具有应用容器的所有字段，除了 readinessProbe
		# 因为 Init 容器无法定义不同于完成<completion> 的就绪<readiness> 之外的其他状态
		# 这会在验证过程中强制执行
		
	# 在 Pod 中的每个 app 和 Init 容器的名称必须唯一
		# 与其他任何容器共享同一个名称，会在验证时抛出错误
```

```shell
# 容器探针
	# 探针是由 kubelet 对容器执行的定期诊断
		# 要执行诊断，kubelet 调用由容器实现的 Handler
		# 三种类型的处理程序
			# ExecAction : 容器内执行指定命令，命令退出返回码为 0 时，则认定为诊断成功
			# TCPSocketAction : 对指定端口上的容器的 IP 地址进行 TCP 检查，如果连接成功，诊断成功
			# HTTPGetAction	: 对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求
				# 如果响应的 200 <= 状态码 < 400，则诊断成功
               
        # 每次探测都将获得以下三种结果之一
        	# 成功、失败、未知
        	
# 探测方式
	# livenessProbe : 指示容器是否正在运行，如果存活探测失败，则 kubelet 杀死容器
		# 并且容器将受其 重启策略 的影响
		# 如果容器不提供存活探针，默认状态为 Success
		
	# readinessProbe : 指示容器是否准备好服务请求
		# 如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址
		# 初始延迟之前的就绪状态默认为 Failure
		# 如果容器不提供就绪探针，则默认状态为 Success
```

```shell
# Start、Stop 与上述类似，都是在 yaml 文件中对 spec 下的子属性做设置
# Pod phase 可能存在的值
	# 挂起(Pending) : Pod 已被 K8S 系统接受，但有一个或多个容器镜像尚未创建
		# 等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间
		
	# 运行中(Running) : 该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建
		# 至少有一个容器正在运行，或者正处于 启动或重启 状态
		
	# 成功(Succeeded) : Pod 中的所有容器都被成功终止，并且不会再重启
	
	# 失败(Failed) : Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止
		# 也就是说，容器以非 0 状态退出或者被系统终止
		
	# 未知(Unknown) : 因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败
```

## 资源控制器

```shell

```

