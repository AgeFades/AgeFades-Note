[TOC]

# 硅谷 - Docker实战

## 一. 基本概念

- Go语言编写，使用 `名称空间` 提供 容器的隔离工作区
- 运行容器时，Docker为该容器创建一组 名称空间（提供隔离）
  - 每个容器都在单独的名称空间中运行
  - 并且，对其的访问权限仅次于该名称空间

### Docker架构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637896260642.png)

- K8S：CRI（Container Runtime Interface）
- Client：客户端（命令行或者界面）
- Docker_Host：Docker主机
- Docker_Daemon：后台进程
- Containers：容器
- Images：镜像
- Registries：仓库，[官方仓库地址](https://hub.docker.com/search)

### Docker隔离原理

#### namespace（资源隔离）

| namespace | 系统调用参数  | 隔离内容                   |
| --------- | ------------- | -------------------------- |
| UTS       | CLONE_NEWUTS  | 主机和域名                 |
| IPC       | CLONE_NEWIPC  | 信号量、消息队列、共享内存 |
| PID       | CLONE_NEWPID  | 进程编号                   |
| Network   | CLONE_NETWORK | 网络设备、网络栈、端口等   |
| Mount     | CLONE_NEWNS   | 挂载点（文件系统）         |
| User      | CLONE_NEWUSER | 用户和用户组               |

#### cgroups（资源限制）

- 主要功能：
  - 资源限制
  - 优先级分配
  - 资源统计
  - 任务控制
- cgroup资源控制系统，每种子系统独立控制一种资源

| 子系统                          | 功能                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| cpu                             | 使用调度程序，控制任务对CPU的使用                            |
| cpuacct(CPU Accounting)         | 自动生成cgroup中，任务对CPU资源使用情况的报告                |
| cpuset                          | 为cgroup中的任务分配独立的CPU(多处理器系统时)和内存          |
| devices                         | 开启或关闭cgroup中，任务对设备的访问                         |
| freezer                         | 挂起或恢复，cgroup中的任务                                   |
| memory                          | 设定cgroup中任务对内存使用量的限定，并生成这些任务对内存资源使用情况的报告 |
| pref_event(Linux CPU性能探测器) | 使cgroup中的任务，可以进行统一的性能测试                     |
| net_cls(Docker未使用)           | 通过等级识别符 来 标记网络数据包，从而允许 Linux 流量监控程序（Traffic Controller）识别从具体 cgroup 中生成 的数据包 |

### Docker安装

[官方安装文档](https://docs.docker.com/engine/install/centos/)

#### CentOS安装示例

1. 移除旧版本

```shell
sudo yum remove docker*
```

2. 设置docker yum源

```shell
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

3. 安装最新docker engine

```shell
sudo yum install docker-ce docker-ce-cli containerd.io
```

4. 安装指定版本docker engine

4.1 在线安装

```shell
#找到所有可用docker版本列表
yum list docker-ce --showduplicates | sort -r
# 安装指定版本，用上面的版本号替换<VERSION_STRING>
sudo yum install docker-ce-<VERSION_STRING>.x86_64 docker-ce-cli- <VERSION_STRING>.x86_64 containerd.io
#例如:
#yum install docker-ce-3:20.10.5-3.el7.x86_64 docker-ce-cli-3:20.10.5-3.el7.x86_64 containerd.io
#注意加上 .x86_64 大版本号
```

4.2 离线安装

[官方安装包下载](https://download.docker.com/linux/centos/7/x86_64/stable/Packages/)

[tar下载官方文档](https://docs.docker.com/engine/install/binaries/#install-daemon-and-client-binaries-on-linux)

```shell
rpm -ivh xxx.rpm

# 可以下载 tar、解压启动即可
```

5. 启动服务

```shell
systemctl start docker
systemctl enable docker
```

6. 镜像加速

- /etc/docker/daemon.json 是Docker的核心配置文件

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
#以后docker下载直接从阿里云拉取相关镜像
```

## 二. Docker命令

[官方命令文档](https://docs.docker.com/engine/reference/commandline/docker/)

| 命令      | 作用                                                         |
| --------- | ------------------------------------------------------------ |
| attach    | 绑定到运行中容器的标准输入、输出、错误流                     |
| build     | 从一个 Dockerfile 文件构建镜像                               |
| commit    | 把容器的改变提交，创建一个薪的镜像                           |
| cp        | 容器和本地文件系统间复制 文件/文件夹                         |
| create    | 创建新容器，但并不启动                                       |
| diff      | 检查容器里文件系统结构的更改。<br>A：添加文件或目录，D：文件或者目录删除，C：文件或目录更改 |
| events    | 获取服务器的实时事件                                         |
| exec      | 在运行时的容器内运行命令                                     |
| export    | 导入容器的文件系统为一个tar文件                              |
| history   | 显示镜像的历史                                               |
| images    | 列出所有镜像                                                 |
| import    | 导入tar的内容，创建一个镜像                                  |
| info      | 显示系统信息                                                 |
| inspect   | 获取docker对象的底层信息                                     |
| kill      | 杀死一个或多个容器                                           |
| load      | 从tar文件加载镜像                                            |
| login     | 登陆Docker registry                                          |
| logout    | 退出Docker registry                                          |
| logs      | 获取容器日志                                                 |
| pause     | 暂停一个或多个容器                                           |
| port      | 列出容器的端口映射                                           |
| ps        | 列出所有容器                                                 |
| pull      | 从 registry 下载一个 image 或 repository                     |
| push      | 给 registry 推送一个 image 或 repository                     |
| rename    | 重命名一个容器                                               |
| restart   | 重启一个或多个容器                                           |
| rm        | 移除一个或多个容器                                           |
| rmi       | 移除一个或多个镜像                                           |
| run       | 创建并启动容器                                               |
| save      | 把一个或多个镜像保存为tar文件                                |
| search    | 去docker hub寻找镜像                                         |
| start     | 启动一个或多个容器                                           |
| stats     | 显示容器资源的实时使用状态                                   |
| stop      | 停止一个或多个容器                                           |
| tag       | 给源镜像创建一个新的标签，变成新的镜像                       |
| top       | 显示正在运行容器的进程                                       |
| unpause   | pause的反操作                                                |
| update    | 更新一个或多个docker容器配置                                 |
| version   | 查看版本                                                     |
| container | 管理容器                                                     |
| image     | 管理镜像                                                     |
| network   | 管理网络                                                     |
| volume    | 管理卷                                                       |

### docker镜像相关

[docker官方仓库](https://hub.docker.com/)

```shell
# 镜像所有管理操作
docker image --help
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637905910021.png)

#### 镜像组成

- 基础环境 + 软件
- 举例：redis 的完整镜像是 linux系统 + redis软件
- alpine：超级经典版的 linux，大小 5mb
  - alpine + redis = 29mb
  - 没有 alpine3 的，就是 centos 基本版
  - 以后自己选择下载镜像的时候，尽量使用 alpine：slim：

### 常见命令

##### 删除全部镜像

```shell
docker rmi -f $(docker images -aq)
```

##### 移除游离镜像

```shell
# dangling:游离镜像(没有镜像名字的)
docker image prune
```

##### 重命名

```shell
docker tag 原镜像:标签 新镜像名:标签
```

##### docker create

- 创建容器

- docker create [设置项] 镜像名 [启动] [启动参数...]
- 举例如下：

```shell
# -p port1:port2，port1 是宿主机端口、port2是容器端口
docker create --name myredis -p 6379(主机的端口):6379(容器的端口) redis
```

##### docker kill

```shell
# 类似 kill -9，强制停机
docker kill
```

##### docker stop

```shell
# 优雅停机，正在运行中的程序处理完所有事情后再停止
docker stop
```

##### docker run

```shell
# 创建并启动容器
# docker run -d == docker create + docker start
# -d 表示后台启动
docker run --name myredis2 -p 6379:6379 redis -d
```

##### docker attach

```shell
# 进容器，绑定的是控制台，可能导致容器停止，不要用这个
docker attach
```

##### docker exec -it

```shell
# 交互方式进容器
# 0用户表示以特权方式进入容器
docker exec -it -u 0:0 --privileged mynginx4 /bin/bash
```

##### docker inspect

```shell
# 查看镜像或容器
docker inspect 容器
docker inspect image /network/volume ...
```

##### docker commit

```shell
# 对容器进行修改后，用 commit 保存最新镜像
# 把新镜像推送到远端 docker hub，方便后来其他机器下载使用
docker commit -a agefades -m "first commit" mynginx4 mynginx:v4
```

##### export/import

```shell
# docker export导出的文件被import导入以后变成镜像，并不能直接启动容器，需要知道之前的启动命令
# (docker ps --no-trunc)，然后再用下面启动
docker run -d -P mynginx:v6 /docker-entrypoint.sh nginx -g 'daemon off;'

# 或者docker image inspect 看之前的镜像，把 之前镜像的 Entrypoint的所有和 Cmd的连接起来就 能得到启动命令
```

##### save/load

```shell
# 把busybox镜像保存成tar文件
docker save -o busybox.tar busybox:latest  

# 把压缩包里面的内容直接导成镜像
docker load -i busybox.tar 
```

##### docker build

```shell
# 根据一个Dockerfile构建出镜像
docker build -t mybusy66:v6 -f Dockerfile .
```

### 容器状态

- Created：新建
- Up：运行中
- Pause：暂停
- Exited：退出

### 典型命令

#### docker run

