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
# CentOS8 需要先执行下面第一条命令
sudo yum -y erase podman buildah

sudo yum -y install docker-ce docker-ce-cli containerd.io
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

### docker-compose安装

- centos示例

```shell
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.1.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
```

## 二. Docker命令

```bash
# 清理docker游离镜像，释放磁盘资源
docker system prune -a
```

[官方命令文档](https://docs.docker.com/engine/reference/commandline/docker/)

- 中文翻译文档，可使用 utools 程序员手册插件、docker命令手册

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637976748251.png)

### docker镜像相关

[docker官方仓库](https://hub.docker.com/)

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

```apl
docker tag 原镜像:标签 新镜像名:标签
```

### 容器状态

- Created：新建
- Up：运行中
- Pause：暂停
- Exited：退出

### Docker镜像的推送操作

- 示例：

```shell
# 拉 nginx 镜像
docker pull nginx

# run 一个容器
docker run -d --name mynginx -p 80:80 nginx

# 提交容器，制作镜像
docker commit -a agefades -m "测试提交" mynginx mynginx:v1

# 查看新镜像
docker images
```

- 在 docker hub 中新建一个 repository

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637978681292.png)

- 命令行登陆 docker hub

```apl
docker login

# 登陆信息保存在 ~/.docker/config.json
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637978929534.png)

- 镜像改名

```apl
docker tag mynginx:v1 agefades/mynginx:v1
```

- 推送

```apl
docker push agefades/mynginx:v1
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637979343007.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637979352610.png)

### 导出与导入

- 导出容器

```apl
docker export -o nginx.tar mynginx
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637980896238.png)

- 导入成镜像

```apl
# 这种方式有坑，有需要用下面的导出镜像
docker import nginx.tar mynginx:v2
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637980961413.png)

- 导出镜像

```apl
docker save -o myginx.tar mynginx:v1
```

- 加载镜像

```apl
docker load -i mynginx.tar
```

### 重启策略

- no：默认策略，在容器退出时不重启容器
- on-failure：容器非正常退出时，重启容器
- on-failure:3：容器非正常退出时，重启容器，最多重启3次
- always：容器退出时总是重启容器
- unless-stopped：容器退出时，总是重启容器，但不考虑 Docker 守护进行启动时，就已经停止的容器

## 三. 网络和存储原理

- docker底层使用自己的 存储驱动
  - 来组建文件内容 storage drivers
  - docker 基于 AUFS（联合文件系统）

### 1. Docker存储

#### 镜像如何存储

```dockerfile
FROM busybox
CMD ping www.baidu.com
```

- docker history 查看镜像分层

```shell
docker history nginx
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637992052379.png)

- docker inspect nginx 查看 nginx 镜像怎么存的

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637992142521.png)

##### LowerDir

- 底层目录，diff（存储不同），包含小型linux和装好的软件

```apl
# 用户文件;
/var/lib/docker/overlay2/67b3802c6bdb5bcdbcccbbe7aed20faa7227d584ab37668a03ff6952e 631f7f2/diff:

# nginx的启动命令放在这里
/var/lib/docker/overlay2/f56920fac9c356227079df41c8f4b056118c210bf4c50bd9bb077bdb4 c7524b4/diff: 

# nginx的配置文件在这里
/var/lib/docker/overlay2/0e569a134838b8c2040339c4fdb1f3868a7118dd7f4907b40468f5fe6 0f055e5/diff: 

# 小linux系统
/var/lib/docker/overlay2/2b51c82933078e19d78b74c248dec38164b90d80c1b42f0fdb1424953 207166e/diff: 
```

##### MergedDir

- 合并目录，容器完整的工作目录全内容，都在合并层
  - 数据卷在容器层产生
  - 所有的增删改都在容器层

##### UpperDir

- 上层目录

##### WorkDir

- 工作目录（临时层），pid

##### 总结

- LowerDir(底层) -> UpperDir -> MergedDir -> WorkDir(临时内容)
- docker底层的 storage driver 完成了以上的目录组织结果

##### Images and layers

- docker 镜像由一系列层组成，每层是 dockerfile 的一条指令
  - 除最后一层外，其余每层都是只读的
- 举例：
  - 每一个指令都可能会引起镜像改变，4个命令创建4个层

```dockerfile
# 基础镜像
FROM ubuntu:15.04
# 复制文件
COPY . /app
# 构建应用程序
RUN make /app
# 容器中执行的命令
CMD python /app/app.py
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637992745805.png)

##### Container and layers

- 容器和镜像之间，主要区别是 `可写顶层`
- 在容器中，所有写操作都存储在 可写层 中
- 删除容器后，可写层也被删除，容器不变

##### 磁盘容量预估

```shell
docker ps -s
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637992876741.png)

- size：每个容器的可写层的数据量（磁盘上）
- virtual size：容器只读层 + 可写层大小

##### 镜像如何挑选

- busybox：一个集成了一百多个最常用 linux 命令和工具的软件
- alpline：基于 busybox 的面向安全的、轻微Linux经典最小镜像
- slim：瘦身镜像
- 优先选择 alpine 类型

##### Copy On Write

- 可写层对只读层修改，先复制一份到可写层，写完再覆盖回去

#### 容器如何挂载

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637993165703.png)

- 容器支持三种挂载方式：
  - volumes：
    - 存储在宿主机磁盘，由docker管理
    - docker持久化存储数据的最佳方式
  - Bind mounts：
    - 可以在 任何地方 存储在主机系统上
    - 都可以对其修改
  - tmpfs mounts：
    - 存储在宿主机内存，并永远不会写入宿主机磁盘

### 2. Docker网络

#### 端口映射

- 举例如下：

```shell
docker create -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 --name hello-mysql
mysql:5.7
```

#### 容器互联

- --link name:alias
  - name：连接容器的名称
  - alias：连接的别名
- 举例如下：

```shell
docker run -d -e MYSQL_ROOT_PASSWORD=123456 --name mysql01 mysql:5.7
docker run -d --link mysql01:mysql --name tomcat tomcat:7
docker exec -it tomcat bash
cat /etc/hosts
ping mysql
```

#### 自定义网络

##### 默认网络原理

- docker 使用 linux 桥接，在宿主机上虚拟一个 docker0（容器网桥）
- 启动容器时，根据docker0的网段随机分配一个ip给容器
- docker0是每个容器的默认网关，容器之间能够互相通信

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637993701980.png)

##### 网络模式

| 网络模式  | 配置                    | 说明                                                         |
| --------- | ----------------------- | ------------------------------------------------------------ |
| bridge    | --net=bridge            | 默认值，docker0为容器创建ip                                  |
| none      | --net=none              | 不配网络，用户自己进容器配                                   |
| container | --net=container:name/id | 多个容器共享network namespace<br>k8s中pod就是多个容器共享一个network namespace |
| host      | --net=host              | 容器和宿主机共享网络                                         |
| 自定义    | --net=自定义网络        | 自建网络，创建容器可指定自建网络，docker-compose中网络就是自定义网络模式 |

## 四. Dockerfile

- 一般分为4部分
  - 基础镜像信息
  - 维护者信息
  - 镜像操作指令
  - 启动时执行指令

| 指令        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| FROM        | 指定基础镜像                                                 |
| MAINTAINER  | 指定维护者信息，已过时<br>使用LABEL maintainer=xxx 代替      |
| RUN         | 运行命令                                                     |
| CMD         | 指定容器启动时的默认命令                                     |
| ENTRYPOINT  | 指定镜像的默认入口运行命令                                   |
| EXPOSE      | 声明镜像内服务监听的端口                                     |
| ENV         | 指定环境变量，可以在docker run的时候用-e改变<br>会被固化到image的config里 |
| ADD         | 复制指定的src路径下的内容到容器中的dest路径下<br>src可以为url会自动下载，可以为tar文件，会自动解压 |
| COPY        | 复制宿主机src路径下的内容，到镜像中的dest路径下              |
| LABEL       | 指定生成镜像的元数据标签信息                                 |
| VOLUME      | 创建数据卷挂载点                                             |
| USER        | 指定运行容器时的用户名或UID                                  |
| WORKDIR     | 工作目录，用于后续的RUN、CMD、ENTRYPOINT                     |
| ARG         | 镜像使用参数，可以在build的时候用--build-args改变            |
| OBBUILD     | 配置当创建的镜像、作为其他镜像的基础镜像时，所指定的创建操作指令 |
| STOPSIGNAL  | 容器退出的信号值                                             |
| HEALTHCHECK | 健康检查                                                     |
| SHELL       | 指定使用shell时的默认shell类型                               |
|             |                                                              |

### FROM

- 指定基础镜像，如 FROM alpine

### LEABL

- 标注镜像说明信息
- 举例如下：

```apl
LABEL multi.label1="value1" multi.label2="value2" other="value3"
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```

### RUN

- 在当前镜像层顶部的新层执行任何命令、并提交结果、生成新的镜像层
  - 新镜像层将用于下一步
  - exec形式可以避免破坏 shell 字符串，并使用不包含指定 shell 可执行文件的、基本镜像运行 run 命令
  - 可以用 SHELL 命令更改 shell 的默认形式
  - 在 shell 形式中，可使用 \ 将一条RUN指令执行到下一条

```apl
# shell 形式，/bin/sh -c 的方式运行，避免破坏 shell 字符串
RUN <command>

# exec 形式
RUN ["executable", "param1", "param2"]
```

- 举例：

```dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'

#上面等于下面这种写法
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

RUN ["/bin/bash", "-c", "echo hello"]
```

```dockerfile
# 测试案例
FROM alpine

LABEL maintainer=agefades xx=aa

ENV msg='hello'

RUN echo $msg

RUN ["echo","$msg"]

RUN /bin/sh -c 'echo $msg'

RUN ["/bin/sh","-c","echo $msg"]

CMD sleep 10000
#总结; 由于[]不是shell形式，所以不能输出变量信息，而是输出$msg。其他任何/bin/sh -c 的形式都 可以输出变量信息
```

- shell：/bin/sh -c <command> 的方式
- exec ["/bin/sh", "-c", command] 的方式 == shell方式
  - exec 默认方式不会进行变量替换

### CMD和ENTRYPOINT

#### 都可以作为容器启动入口

- CMD的三种写法：
  - CMD ["executable", "param1", "param2"]
    - exec方式，首选方式
  - CMD ["param1", "param2"]
    - 为ENTRYPOINT提供默认参数
  - CMD command param1 param2
    - shell 形式
- ENTRYPOINT的两种写法：
  - ENTRYPOINT ["executable", "param1", "param2"]
    - exec方式，首选方式
  - ENTRYPOINT command param1 param2
    - shell 形式
- 举例如下：

```dockerfile
# 一个示例
FROM alpine
LABEL maintainer=agefades
CMD ["1111"]
CMD ["2222"]
ENTRYPOINT ["echo"]
#构建出如上镜像后测试
docker run xxxx:效果 echo 2222
```

#### 只能有一个CMD

- Dockerfile只能有一条CMD指令，多个则只有最后一个生效
- CMD主要是为执行中的容器提供默认值
  - 可包含 可执行文件
  - 也可省略 可执行文件，这种情况下，必须指定 ENTRYPOINT

#### CMD为ENTRYPOINT提供默认参数

- 如果CMD为ENTRYPOINT提供默认参数，
  - 则CMD和ENTRYPOINT 均应使用 JSON 数据格式指定

#### docker run启动参数会覆盖CMD内容

```dockerfile
# 一个示例
FROM alpine
LABEL maintainer=agefades
CMD ["1111"]
ENTRYPOINT ["echo"]
#构建出如上镜像后测试
docker run xxxx:什么都不传则 echo 1111 
docker run xxx arg1:传入arg1 则echo arg1
```

### ARG和ENV

#### ARG

- 定义变量，可在构建时使用 --build-arg= 传递
- 同名参数会覆盖Dockerfile中 ARG 指令
- 传递Dockerfile未定义的参数，构建输出警告
- ARG只在构建期有效，运行期无效
- 从定义的变量行开始生效
- 使用ENV定义的变量，使用覆盖ARG的同名变量

#### ENV

- 构建阶段中，所有后续指令的环境中使用，可以内联替换
- 引号和反斜杠可用于在值中，包含空格

```dockerfile
ENV MY_MSG hello
ENV MY_NAME="John Doe"
ENV MY_DOG=Rex\ The\ Dog
ENV MY_CAT=fluffy
#多行写法如下
ENV MY_NAME="John Doe" MY_DOG=Rex\ The\ Dog \
MY_CAT=fluffy
```

- docker run --env 可以修改这些值
- 容器运行时，ENV值生效
- ENV在 image 阶段会被解析并持久化

```dockerfile
FROM alpine
ENV arg=1111111
ENV runcmd=$arg
RUN echo $runcmd
CMD echo $runcmd
```

#### 综合案例

```dockerfile
FROM alpine
ARG arg1=22222
ENV arg2=1111111
ENV runcmd=$arg1
RUN echo $arg1 $arg2 $runcmd
CMD echo $arg1 $arg2 $runcmd
```

### ADD和COPY

#### COPY

- 两种写法

```dockerfile
COPY [--chown=<user>:<group>] <src>... <dest>

COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]
```

- --chown 仅在用于构建 Linux 的Dockerfile上支持
- 从 src 宿主机复制 文件或目录 到 容器中、路径为 dest
- 可指定多个 src，但文件和目录的路径，将被解释为相对于构建上下文的源
- 每个 src 都可以包含通配符，并且匹配将使用 Go 的 filepath.Match 规则进行

```dockerfile
COPY hom* /mydir/ #当前上下文，以home开始的所有资源

COPY hom?.txt /mydir/ # ?匹配单个字符

COPY test.txt relativeDir/ # 目标路径如果设置为相对路径，则相对与WORKDIR 开始 # 把 “test.txt” 添加到 <WORKDIR>/relativeDir/
	
COPY test.txt /absoluteDir/ #也可以使用绝对路径，复制到容器指定位置

#所有复制的新文件都是uid(0)/gid(0)的用户，可以使用--chown改变 COPY --chown=55:mygroup files* /somedir/
COPY --chown=bin files* /somedir/

COPY --chown=1 files* /somedir/

COPY --chown=10:11 files* /somedir/
```

#### ADD

- 同COPY用法， 不过自带远程下载、或解压功能
- 注意：
  - src必须在构建的上下文中
  - src是url，并dest不以斜杠结尾，则从url下载文件并复制到dest
    - 如果dest以斜杠结尾，则自动推断出url名字（保留最后一部分）保存到dest
  - src是目录，则复制目录的整个内容，包括文件系统元数据

### WORKDIR和VOLUME

#### WORKDIR

- 为 RUN、CMD、ENTRYPOINT、COPY、ADD 设定工作目录
  - 如果 workdir 不存在，即使后面的指令中未使用它也将被创建
- workdir可在dockerfile多次使用，如果提供了相对路径
  - 则它为相对于上次 workdir 的路径

```dockerfile
WORKDIR /a 
WORKDIR b 
WORKDIR c 
RUN pwd #结果 /a/b/c
```

- 也可以用环境变量

```dockerfile
ENV DIRPATH=/path
WORKDIR $DIRPATH/$DIRNAME RUN pwd
#结果 /path/$DIRNAME
```

#### VOLUME

- 把容器的文件映射到宿主机

```dockerfile
VOLUME ["/var/log/"] #可以是JSON数组 VOLUME /var/log #可以直接写
VOLUME /var/log /var/db #可以空格分割多个
```

### USER

- 写法：

```dockerfile
USER <user>[:<group>]
USER <UID>[:<GID>]
```

- 设置运行镜像的用户名、或UID、及可选的用户组、或GID

### EXPOSE

- 指定监听端口，支持 TCP 和 UDP，默认TCP

```dockerfile
EXPOSE <port> [<port>/<protocol>...]
EXPOSE [80,443]
EXPOSE 80/tcp
EXPOSE 80/udp
```

### multi-stage builds

- 多阶段构建
- [官方文档](https://docs.docker.com/develop/develop-images/multistage-build/)

```dockerfile
### 我们如何打包一个Java镜像 FROM maven
WORKDIR /app
COPY . .
RUN mvn clean package
COPY /app/target/*.jar /app/app.jar
ENTRYPOINT  java -jar app.jar
## 这样的镜像有多大? ## 我们最小做到多大??
```

#### 生产案例

```dockerfile
#以下所有前提 保证Dockerfile和项目在同一个文件夹 
# 第一阶段:环境构建; 用这个也可以
FROM maven:3.5.0-jdk-8-alpine AS builder WORKDIR /app
ADD ./ /app
RUN mvn clean package -Dmaven.test.skip=true
# 第二阶段，最小运行时环境，只需要jre;
# 第二阶段并不会有第一阶段哪些没用的层 
# 基础镜像没有 jmap; jdk springboot-actutor(jdk)
FROM openjdk:8-jre-alpine
LABEL maintainer="534096094@qq.com"
# 从上一个阶段复制内容
COPY --from=builder /app/target/*.jar /app.jar
# 修改时区
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone && touch /app.jar
ENV JAVA_OPTS=""
ENV PARAMS=""
# 运行jar包
ENTRYPOINT [ "sh", "-c", "java -Djava.security.egd=file:/dev/./urandom $JAVA_OPTS -jar /app.jar $PARAMS" ]
```

##### pom

```xml
<!--为了加速下载需要在pom文件中复制如下 --> 
<repositories>
    <repository>
        <id>aliyun</id>
        <name>Nexus Snapshot Repository</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <layout>default</layout>
        <releases>
            <enabled>true</enabled>
        </releases>
<!--snapshots默认是关闭的,需要开启 --> <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </repository>
</repositories>
<pluginRepositories>
    <pluginRepository>
        <id>aliyun</id>
        <name>Nexus Snapshot Repository</name>
        <url>https://maven.aliyun.com/repository/public</url>
        <layout>default</layout>
        <releases>
            <enabled>true</enabled>
        </releases>
        <snapshots>
            <enabled>true</enabled>
        </snapshots>
    </pluginRepository>
</pluginRepositories>
```

##### 镜像时间同步细节

```dockerfile
# 小细节
RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo
   'Asia/Shanghai' >/etc/timezone
# 或者
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
可以让镜像时间同步。

# 容器同步系统时间 CST(China Shanghai Timezone)
-v /etc/localtime:/etc/localtime:ro
# 已经不同步的如何同步?
docker cp /etc/localtime 容器id:/etc
```

#### 自动拉代码并构建镜像

```bash
docker build --build-arg url="git address" -t demo:test .
```

```dockerfile
FROM maven:3.6.1-jdk-8-alpine AS buildapp
# 第二阶段，把克隆到的项目源码拿过来
# COPY --from=gitclone * /app/ WORKDIR /app
COPY pom.xml .
COPY src .
# /app 下面有 target
RUN mvn clean package -Dmaven.test.skip=true 
RUN pwd && ls -l
RUN cp /app/target/*.jar /app.jar
RUN ls -l
# 以上第一阶段结束，我们得到了一个 app.jar
# 只要一个JRE
# FROM openjdk:8-jre-alpine
FROM openjdk:8u282-slim
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
LABEL maintainer="534096094@qq.com"
# 把上一个阶段的东西复制过来
COPY --from=buildapp /app.jar /app.jar
# docker run -e JAVA_OPTS="-Xmx512m -Xms33 -" -e PARAMS="--spring.profiles=dev - -server.port=8080" -jar /app/app.jar
# 启动java的命令
ENV JAVA_OPTS=""
ENV PARAMS=""
ENTRYPOINT [ "sh", "-c", "java -Djava.security.egd=file:/dev/./urandom
$JAVA_OPTS -jar /app.jar $PARAMS" ]
```

### Images瘦身实践

- 选择最小基础镜像
- 合并RUN环境的所有指令，少生成一些层
- RUN期间可能安装其他程序，会生成临时缓存，要自行删除

- 举例如下：

```dockerfile
# 开发期间，逐层验证正确的 RUN xxx
RUN xxx
RUN aaa \
aaa \ vvv \

# 生产环境
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion \
  && rm -rf /var/lib/apt/lists/*
```

- 使用 `.dockerignore` 文件，排除上下文无需参与构建的资源
- 使用多阶段构建
- 合理使用构建缓存加速构建，[--no-cache]

### SpringBoot Dockerfile示例

```dockerfile
FROM openjdk:8-jre-alpine
LABEL maintainer="534096094@qq.com"
COPY target/*.jar /app.jar
RUN  apk add -U tzdata; \
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime; \
echo 'Asia/Shanghai' >/etc/timezone; \
touch /app.jar;
ENV JAVA_OPTS=""
ENV PARAMS=""
EXPOSE 8080
ENTRYPOINT [ "sh", "-c", "java -Djava.security.egd=file:/dev/./urandom
$JAVA_OPTS -jar /app.jar $PARAMS" ]
# 运行命令 docker run -e JAVA_OPTS="-Xmx512m -Xms33 -" -e PARAMS="-- spring.profiles=dev --server.port=8080" -jar /app/app.jar
```

## 五. docker-compose

- 用 yaml 语法，一次创建并启动多个容器，通过自定义网络使这些容器网络互通
- 启动/停止命令，需要在 docker-compose.yaml 同级目录
  - 非默认文件名(docker-compose.yaml)，需要 -f 参数指定文件

```bash
# 启动
docker-compose up -d
# 停止
docker-compose down
```

- 举例如下：

```yaml
# 指定版本号;查看文档https://docs.docker.com/compose/compose-file/
version: "3.9" 
# 所有需要启动的服务
services: 
      # 第一个服务的名字
      web: 
            # docker build -t xxx -f Dockerfile .
            build: 
                  dockerfile: Dockerfile
                  context: .
            # 镜像名
            image: 'hello:py'
            # 指定启动容器暴露的端口
            ports: 
                  - "5000:5000" 
      # 第二个服务的名字
      redis: 
            image: "redis:alpine" 
      # mysqlserver: #第三个服务
```

```yaml
version: "3.7"
services:
  app:
    image: node:12-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos
    networks:
      - hello
      - world
    deploy: #安装docker swarm
      replicas: 6 #指定副本:处于不同的服务器(负载均衡+高可用)
  mysql: #可以代表一个容器，ping 服务名 mysql 可以访问
    image: mysql:5.7 #负载均衡下，数据一致怎么做???主从同步，读写分离 
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos
    networks: #这个服务加入那个自定义网络
      - hello
    deploy: #安装docker swarm
      replicas: 6 #指定副本:处于不同的服务器(负载均衡+高可用)
  redis:
    image: redis
    networks:
      - world
volumes:
  todo-mysql-data:
networks:
   hello:
   world:
```

