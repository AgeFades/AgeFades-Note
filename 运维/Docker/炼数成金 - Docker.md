# 整合 - Docker

## Docker 的技术原理介绍

### 	cgroups

​		Controller Group ，**限制某个或者某些进程的分配资源**

​		在该 group 中，有分配好的特定比例的 cpu 时间、IO 时间、可用内存大小等

​		cgroups 是将任意进程进行分组化管理的 Linux 内核功能

#### 		重要概念

##### 			子系统

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

### ​	App 打包

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

### ​	Registry | Hub

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

## Docker 容器互联

