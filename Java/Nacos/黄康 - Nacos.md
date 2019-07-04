# 注意版本问题

如果采用Boot2.1.X以及Cloud [Greenwich](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies/Greenwich.SR1)版请安装1.0.0以上的版本（包括1.0.0）

如果采用Boot2.0.X以及Cloud  [Finchley](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies/Finchley.SR2) 版请安装1.0.0以下的版本（不包括1.0.0）

如果采用Boot1.5以及以下请安装更低版本

# 如何安装Nacos

​	首先我们要先去下载nacos

​	https://github.com/alibaba/nacos/releases

​	这里是nacos的下载路径

​	下载好之后我们解压到目录下

# 运行

## 在windows的环境下

​	找到目录的bin目录，里面有个startup.cmd这个文件，双击执行，然后出现cmd控制台窗口此时不要关闭

​	![](img\nacos-server.png)

然后我们可以看得到他的console路径，复制下来放到浏览器上，进入浏览器，默认账户密码都为nacos，

![](img\nacos-login.png)

然后登陆

![](img\nacos-login-end.png)

这样我们就运行好了

## Linux环境

找到当前目录，进入bin目录下执行sh startup.sh -m standalone

![](img\linux-nacos.png)

然后等待启动

![](img\linux-nacos-2.png)

然后也能直接访问了

后台启动使用nohup sh startup.sh -m standalone &

![](img\nohup-nacos.png)



# Docker版

## 单机

### 快速启动

docker一键运行

```
docker run -d --name nacos -e MODE=standalone -p 8848:8848 docker.io/nacos/nacos-server:1.0.0
```

### 生产启动

！！！！！！！！！！！！！(注意现在全部采用Nacos1.0.0为兼容SpringClou以及Boot2.1)

首先下载mysql配置mysql的地址，先安装mysql（MYSQL注意使用，5.7）

首先创建文件夹以及挂载目录。将配置文件以及数据文件持久化出来

```
mkdir -p /docker/nacos/conf
mkdir -p /docker/nacos/data
```

然后进入mysql配置文件加入

```
vim /docker/nacos/conf/my.cnf

加入下面内容
[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

然后启动mysql容器

```
docker run -p 3306:3306 \
--name nacos-mysql \
-e MYSQL_ROOT_PASSWORD=bigkang \
--privileged=true \
-v /docker/nacos/data:/var/lib/mysql \
-v /docker/nacos/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

然后我们进入容器，创建数据库

```
docker exec -it nacos-mysql bash

mysql -u root -p
输入上面的密码bigkang
然后创建数据库
create database nacos;
```

然后我们给数据库添加数据，我这里采用的是bigkang作为name，MySQL数据在旁边的标题点击一下

这样MySQL就已经搭建完毕了，我们来搭建Nacos,这里注意一定要用自己的MySQL的ip地址

```
首先创建配置文件
vim /docker/nacos/conf/application.properties
将下面内容粘贴进去，注意这里用户和密码以及mysql的ip地址都要修改

# spring
server.servlet.contextPath=${SERVER_SERVLET_CONTEXTPATH:/nacos}
server.contextPath=/nacos
server.port=${NACOS_SERVER_PORT:8848}
spring.datasource.platform=${SPRING_DATASOURCE_PLATFORM:""}
nacos.cmdb.dumpTaskInterval=3600
nacos.cmdb.loadDataAtStart=false


spring.datasource.platform=mysql
db.num=1
# 注意修改自己IP重要的话重复说
db.url.0=jdbc:mysql://localhost:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user=root
db.password=bigkang


server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D
# default current work dir
server.tomcat.basedir=
## spring security config
### turn off security

nacos.security.ignore.urls=/,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/login,/v1/console/health/**,/v1/cs/**,/v1/ns/**,/v1/cmdb/**,/actuator/**,/v1/console/server/**
# metrics for elastic search
management.metrics.export.elastic.enabled=fasle
management.metrics.export.influx.enabled=false

nacos.naming.distro.taskDispatchThreadCount=10
nacos.naming.distro.taskDispatchPeriod=200
nacos.naming.distro.batchSyncKeyCount=1000
nacos.naming.distro.initDataRatio=0.9
nacos.naming.distro.syncRetryDelay=5000
nacos.naming.data.warmup=true
```

然后我们启动容器,

```
docker run -p 8848:8848 \
--name nacos-server \
-e MODE=standalone \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-d docker.io/nacos/nacos-server:1.0.0




docker run -p 8848:8848 \
--name nacos-server \
-e MODE=standalone \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-d docker.io/nacos/nacos-server:1.0.0
```

然后访问ip:8848/nacos/index.html

## 集群

和单机版一样步骤，但是删除掉-e MODE=standalone,我这里由于测试内存不够，默认启动集群jvm为2g修改为512m，如果内存足够请删除第四行

```
docker run -p 8848:8848 \
--name nacos-node1 \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-v /docker/nacos/conf/docker-startup.sh:/home/nacos/bin/docker-startup.sh \
-d docker.io/nacos/nacos-server:1.0.0

docker exec -it nacos-node1 bash

39.108.158.33:8848
39.108.158.33:8849
111.67.198.232:8850


docker run -p 8849:8848 \
--name nacos-node2 \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-v /docker/nacos/conf/docker-startup.sh:/home/nacos/bin/docker-startup.sh \
-d docker.io/nacos/nacos-server:1.0.0


docker run -p 8850:8848 \
--name nacos-node3 \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-v /docker/nacos/conf/docker-startup.sh:/home/nacos/bin/docker-startup.sh \
-d docker.io/nacos/nacos-server:1.0.0



```

但是启动之后我们还需要去容器中编辑它的配置文件

```
docker exec -it nacos-node1 bash
cd conf
vim cluster.conf

这里设置新的集群地址
例如,一行一个集群地址ip:端口，然后我们就能使用集群了

39.108.158.30:8841
39.108.158.30:8842
39.108.158.30:8843
```

```
注意事项：
	1、如果没有添加集群地址则当前的服务不可用
	2、修改后请将当前的容器提交为新的镜像，否则每次重启集群文件会被刷新掉
```




# 集群版本搭建

## Linux集群搭建

首先我们在一个目录下新建一个集群文件夹

```
mkdir nacos-cluster
```

然后解压三个nacos的文件放进去分别命名为nacos1，nacos2，nacos3

如下图所示（别问如何解压和命名，再问自杀）

![](img\nacos123.png)

然后我们先来将数据库新建好（集群版基于数据库）

首先新建一个名叫nacos的数据库

![](img\nacos-mysql.png)

然后在里面执行sql文件，后面的用户名可以更改，密码采用的是加密算法需要去重新生成加密算法密码然后添加

# MySQL数据

密码默认为nacos(默认加密的算法)

数据库修改之后修改配置文件

进入他的conf目录下找到

 cluster.conf.example这里只是一个模板我们把它复制一下然后改个名字,将他改为cluster.conf

```
cp cluster.conf.example cluster.conf
```

然后进入编辑修改集群地址，（最少三个）(这里配成我们的ip和端口号)

```
vim cluster.conf
```

![](img\cluster集群.png)

然后去修改他的mysql路径

首先编辑配置文件application.properties

```
vim application.properties
```

然后在里面加上mysql的配置信息（路径和用户名密码写自己的还有注意数据库名一致）(三个都要改还有端口)

```
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://39.108.158.33:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user=root
db.password=bigkang
```

然后修改他的端口号（因为我们在一台机器中进行集群所以需要修改端口号）

分别修改

```
nacos1的配置文件
server.port=8801

nacos2的配置文件
server.port=8802

nacos3的配置文件
server.port=8803
```

这样我们就编写好了集群然后我们来启动吧

直接进入bin目录下找到启动脚本直接启动

```
 sh startup.sh 
```

我们一个一个地启动就完成了（注：可能会出现很多问题）

# 集群问题

​	 首先是秒退那么肯定就是Jvm的内存不够了，由于集群模式默认的启动大小是2个g所以我们需要去进行堆内存的调整具体如下

​	![](img\集群堆内存.png)

我们可以看到单机版启动512m，集群的将他已经修改默认是2g，我们把它修改为

```
-server -Xms256m -Xmx512m  -XX:PermSize=128m -XX:MaxPermSize=256m
```

然后再启动就行了



​	如果还是有问题那么我们就先去查看他的日志文件

​	进入logs然后查看文件

```
vim nacos.log
```

这里我们看到了

![](img\数据源错误.png)

这个原因可能是因为数据源没写对，还有就是使用了8.0的数据库作为nacos的数据存储，只要将他改成5.6或者修改mysql8的一些配置文件就能实现

如果出现了各种错误首先我们先排查两个日志文件

```
vim nacos.log 

vim start.out 
```

通过这两个日志文件我们就能定位他的错误了

```

```



# 环境搭建完毕一键清理

## 清理单机版生产级Docker

停止以及删除容器

```
docker stop nacos-server
docker stop nacos-mysql

docker rm nacos-server
docker rm nacos-mysql
```

删除镜像（可以不用删除留着以后用）

```
docker rmi docker.io/nacos/nacos-server
docker rmi docker.io/mysql:5.7
```

删除挂载目录（删除后不可恢复）

```
rm -rf /docker/nacos
```

重新快速部署

```
docker run -p 3306:3306 \
--name nacos-mysql \
-e MYSQL_ROOT_PASSWORD=bigkang \
--privileged=true \
-v /docker/nacos/data:/var/lib/mysql \
-v /docker/nacos/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7


--------------------------------------------------

docker run -p 8848:8848 \
--name nacos-server \
-e JAVA_TS='-Xmx256m -Xms256m -XX:+UseG' \
-e MODE=standalone \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-d docker.io/nacos/nacos-server
```

## 清理集群版生产级Docker

停止容器

```
docker stop  nacos-node1
docker stop  nacos-node2
docker stop  nacos-node3
```

删除容器

```
docker rm  nacos-node1
docker rm  nacos-node2
docker rm  nacos-node3
```

