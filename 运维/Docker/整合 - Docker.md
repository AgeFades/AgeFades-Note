[TOC]

# 整合 - Docker

## Docker 的技术原理介绍

### 	cgroups

​		Controller Group ，**限制某个或者某些进程的分配资源**

​		在该 group 中，有分配好的特定比例的 cpu 时间、IO 时间、可用内存大小等

​		cgroups 是将任意进程进行分组化管理的 Linux 内核功能

#### 			重要概念

##### 					子系统

​				资源控制器，**每种子系统就是一个资源的分配器**

​				例 : CPU 子系统是控制 CPU 时间分配的

​				**首先挂载子系统，然后才有 cgroups**

​				比如先挂载 memory 子系统，然后在其中创建一个 cgroup 节点

​				**在该节点，将需要控制的进程 id 写入，并将控制的属性写入，完成内存的资源限制**

### 	LXC

​		Linux Containers ，**基于容器的操作系统层级的虚拟化技术**

​		借助于 namespace 的隔离机制和 cgroup 限制功能，LXC 提供统一 API 和工具来建立和管理容器

​		LXC 最大优势在于，被整合进内核，而不用单独为内核打补丁

#### 		与其他虚拟化技术的比较

​			性能方面： LXC>>KVM>>XEN

​			内存利用率： LXC>>KVM>>XEN

​			隔离程度： XEN>>KVM>>LXC

### 	AUFS

​		**透明覆盖一或多个现有文件系统的层状文件系统**

​		支持将不同目录挂载到同一个虚拟文件系统下，将不同目录联合在一起，组成一个单一的目录

​		是一种虚拟的文件系统，文件系统不用格式化，直接挂载即可

​		Docker 一直在用 AUFS 作为容器的文件系统

​		当一个进程需要修改一个文件时，AUFS 创建该文件的一个副本

​		AUFS 将多层合并成文件系统的单层表示，该过程被称为 **写入复制**

### 	App 打包

​		LXC 的基础上，Docker 额外提供的 Feature 包括 : 标准统一的打包部署运行方案

​		为了最大化重用 Image，Docker Container 的运行环境实际上，是具有依赖关系的多个 Layer 组成	

​		例 : 一个 apache 运行环境可能是在基础的 rootfs image 的基础上，叠加了包含例如 Emacs 等各种

​		有了层级的 image 做基础，理想中，不同的 APP 就尽可能的共用底层文件系统、相关依赖工具等

​		同一个 App 不同实例也可以实现共用绝大部分数据，进而以 写入复制 的形式维护自己修改的数据

## Docker 的基本概念

### 	Image

​		极度精简的 Linux 程序运行环境

​		是需要定制化 Build 的一个安装包，包括基础镜像 + 应用的二进制部署包

​		Image 内不建议有运行期需要修改的配置文件

​		Dockerfile 用来创建一个自定义的 image

### 	Container

​		是 Image 的实例，共享内核

​		可以运行不同 OS 的 Image，比如 Ubuntu Or CentOS

​		Container 不建议内部开启一个 SSHD 服务

​		Container 没有 IP ，通常不会有服务端口暴露

### 	Daemon

​		创建和运行 Container 的 Linux 守护进程，Docker 最核心的组件

​		可以理解为 Container 的 Container

​		Daemon 可以绑定本地端口并提供 Rest Api 服务，用来远程访问和控制

### 	Registry | Hub

​		Docker 镜像仓库

## Docker 的部署安装

### 	Ubuntu

```shell
uname -a # 检查系统内核版本,docker Linux 内核版本有要求

ls -l /sys/class/misc/device-mapper # 检查驱动

# 安装 Ubuntu 维护的版本，版本低
sudo apt-get install docker.io
source /etc/bash_completion.d/docker.io

# 安装 Docker 维护的版本
sudo apt-get install -y curl # whereis curl 查看系统是否存在 curl，不存在则安装
curl -sSL https://get.docker.com/ubuntu/ | sudo sh
docker version # 查看 docker 版本

# 使用非 root 用户
sudo groupadd docker
sudo gpasswd -a ${USER} docker
sudo service docker restart

```



## Docker 配置文件与日志

### 	配置文件

```shell
cd /etc/sysconfig/docker # docker 配置文件目录

# 重要参数的解释
OPTIONS # 用来控制 Docker Daemon 进程参数
-H # 表示 Docker Daemon 绑定的地址 
--registry-mirror # 表示 docker 镜像残酷地址
--insecure-registry # 表示 私有仓 地址
--selinux-enabled # 是不开启 SELinux，默认开启
--bip # 表示网桥 docker0 使用指定 CIDP 网络地址
-b # 表示采用已经创建好的网桥
http_proxy=xxx:8080 # 代理设置 or https

cd /user/lib/systemd/system/docker.service # docker 配置文件

/var/lib/docker # docker 文件存储位置
/var/lib/docker/mnt # 存储 docker 镜像文件

vim /etc/default/docker # 修改 docker 默认仓库地址配置文件
DOCKER_OPTS="--registry-mirror=" # 可为阿里云 docker 加速器
```

### 	日志

```shell
cd /var/log/message # docker 日志文件写入地址
tail -f /var/log/message | grep docker # 监听 docker 日志
```

## Docker 基础命令讲解

```shell
docker search # 查找 docker 镜像
docker pull # 拉取镜像
docker push # 推送镜像
docker images # 查看本地所有镜像
docker run # 运行镜像
docker ps # 查看正在运行的 docker 容器
docker inspect 容器名 # 返回容器所有信息
docker rm 容器名 # 删除停止的容器
docker logs [-f] [-t] [--tail] 容器名 # 查看容器日志,-f 跟踪 -t 在返回的结果上加上时间戳
docker top 容器名 # 查看容器内进程
docker stop 容器名 # 停止容器(等待)
docker kill 容器名 # 杀死容器(立即)
docker info # 查看 Docker 信息
docker rmi # 删除镜像

docker run [OPTIONS] IMAGE[:tag] [COMMAND] [ARG...]
# 决定容器的运行方式、前台执行还是后台执行
# docker run 后面追加 -d，容器将已后台模式运行
# docker exec 进入到该容器，或者 attach 重新连接容器的会话
# 进行交互操作(Shell)，必须使用 -i -t 参数同容器进行交互
# docker run 时没有指定 --name，daemon 会自动生成一个随机字符串 UUID
# docker 有自动化需求时，将 containerID 输出到指定文件中 : --cidfile="..."
# docker 容器没有特权，不能在容器中再启动一个容器
# docker 默认不能访问其他设备，通过 privileged ，容器就拥有了访问其他设备的权限

docker create/start/stop/pause/unpause
# docker 容器生命周期相关指令
```

## Docker 自定义容器镜像

```shell
docker commit <container> [repo:tag]
# docker 在容器中的操作，如果不做 commit 保存，重启后所有操作消息

vim Dockerfile # 编辑 dockerfile 文件，制作镜像
docker build -t xx . # 通过当前目录下的 dockerfile 构建镜像
```

![UTOOLS1564032018169.png](https://i.loli.net/2019/07/25/5d393c1810bc085036.png)

## Dockerfile

```shell
# KV 形式，所有 V 都是举例
FROM CentOS # 基础镜像

LABEL maintainer="agefades@qq.com" # 作者
LABEL version="1.0" # 版本
LABEL description="Agefades's demo dockerfile" # 描述

# RUN ，镜像构建执行的 Shell 命令，避免过多的 AuFS 文件层，尽量合并成一条命令
RUN yum update && yum install -y vim \
python-dev # 反斜线 \ 换行 

# 尽量使用 WORKDIR,而不是 RUN cd，尽量使用绝对目录
WORKDIR /root # 工作目录，如果没有会自动创建

# ADD，不仅可以添加，还可以解压
ADD hello / # 将本地 hello 文件添加到容器中根目录
ADD test.tar.gz / # 添加到根目录并解压

# COPY 相较于 ADD ，大部分情况下优先级更高，但是没有解压功能
# 添加远程文件或目录，尽量使用 curl 或者 wget

# ENV
ENV MYSQL_VERSION 5.6 # 设置环境变量
RUN apt-get install -y mysql-server="${MYSQL_VERSION}" \
&& rm -rf /var/lib/apt/lists/* # 引用环境变量

# CMD 设置容器启动后默认执行的命令和参数
# 如果 docker run 指定了其他命令，CMD 命令被忽略
# 如果定义多个 CMD，只有最后一个会执行

# ENTRYPOINT 设置容器启动时运行的命令
# 不会被忽略，一定会执行

# ESPOSE 暴露端口
```

## Docker Hub

​	官网 : hub.docker.com

```shell
docker login # 登录账户
docker push xx # 推送镜像，参数为镜像名
```

## Docker 网络

```shell
docker network ls # 展示 docker 网络协议
```

### 	Bridge Network

![UTOOLS1564650535344.png](https://i.loli.net/2019/08/01/5d42ac27aa07625379.png)

### 	容器之间的 link

```shell
link 是一个参数
在 docker 容器内部 iptables 加了一层 DNS 记录
通过其他容器的 name 指向该容器的 ip
```

### 	容器的端口映射

```shell
-p 80:80 # 映射宿主机与 docker 容器的端口关系
```

### 	host

```shell
容器与宿主机共享Ip + Port
```

### 	none

```shell
没有网卡设备
```

## Docker Volume

```shell
-v # 指定宿主机与容器之间的目录 or 文件挂载关系
```

## Docker Compose

```shell
# docker 的一个批处理文件工具
# 通过一个 yml 文件定义多容器的 docker 应用
# 通过一条命令，根据 yml 定义去创建或管理多个容器
```

### 	Services

```shell
# 一个service 代表一个 container
# container 可以从 image 创建，也可以由自定义 dockerfile 构建
# Service 启动类似 docker run，可以指定各种参数
```

```yaml
services:
	db:
		image: postgres:9.4
		volumes:
			- "db-data:/var/lib/postgresql/data"
		networks:
			- back-tier
```

```yaml
services:
	worker:
		build: ./worker
		links:
			- db
			- redis
		networks:
			- back-tier
```

### 	compose 安装

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

docker-compose --version
```

```shell
docker-compose -f docker-compose.yml up -d # 默认文件名可省略 -f 指定文件，-d 后台启动
docker-compose ps
docker-compose stop # 停止不删除容器
docker-compose down # 停止并删除
docker-compose start # 启动容器
docker-compose exec [service_name] bash # 进入服务容器内部

docker network ls # 展示 docker 网络
```

### 	Docker 水平扩展和负载均衡

```shell
docker-compose up --scale [service_name]=10 -d # 将服务容器扩展至10个
```

```yaml
lb:
	image: dockercloud/haproxy
	links:
		- web
	ports:
		- 8080:80
	volumes:
		- /var/run/docker.sock:/var/run/docker.sock
```

