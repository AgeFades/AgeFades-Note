杜超 - Linux 基础开发环境搭建

## 修改 hostname

```shell
# 通过 hostnamectl 修改主机名、静态、瞬态、灵活主机名
hostnamectl set-hostname xxx
hostnamectl --pretty
hostnamectl --static
hostnamectl --transient

# 手动修改 host 主机解析
vim /etc/hosts
# 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
127.0.0.1  xxx
# ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
::1        xxx

# 重启 or 退出连接
reboot
hostnamectl # 查看主机信息
```

## SSH 登录<拒绝密码>

```shell
# 将本机的 id_rsa.pub 复制到服务器中 ~/.ssh/authorized_keys
vim ~/.ssh/authorized_keys

# 更改权限
chown -R 700 ~/.ssh # 在本机也执行一次该命令
chown -R 644 ~/.ssh/authorized_keys

# 修改配置
vim /etc/ssh/sshd_config

# 允许密钥认证,此三项在原配置中被注释，可以直接添加到文件末尾
RSAAuthentication yes
PubkeyAuthentication yes
StrictModes no
# 公钥保存文件
AuthorizedKeysFile .ssh/authorized_keys # 文件默认
# 拒绝密码登录
PasswordAuthentication no # 源文件中最后一行，默认为 yes

# 重启服务
systemctl restart sshd.service

# 此时可以 ssh 登录
sudo ssh -i id_rsa root@xxx # mac 示例，将本机的 id_rsa 复制到 / 路径下就好
```



## Docker 安装

```shell
# 删除旧的 docker 
yum remove docker  docker-client  docker-client-latest  docker-common  docker-latest  docker-latest-logrotate  docker-logrotate  docker-selinux  docker-engine-selinux  docker-engine docker-ce -y

# 删除旧的 docker 文件
rm -rf /var/lib/docker

# 安装所需的依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 安装仓库地址
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 改用阿里源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 查看仓库内可选的版本包
yum list docker-ce --showduplicates | sort -r

# 选择其中一个安装
yum install docker-ce-18.06.1.ce -y

# 启动 docker，并设置开机自启
systemctl start docker
systemctl enable docker
```



## Docker-Compose 安装

```shell
# 官网<翻墙速度慢>
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version

# 国内源安装
pip -V # 检查有没有 python-pip 包

# 没有则安装
yum -y install epel-release 
yum -y install python-pip
pip install --upgrade pip

# 安装 docker-compose
pip install docker-compose
docker-compose -version
```



## Git 安装

```shell
# 安装依赖包
yum install -y curl-devel expat-devel gettext-devel openssl-devel zlib-devel asciidoc gcc perl-ExtUtils-MakeMaker

# 卸载旧的 git
yum remove -y git

# 安装 git
cd /usr/local/src/
wget https://www.kernel.org/pub/software/scm/git/git-2.18.0.tar.xz
tar -vxf git-2.18.0.tar.xz
cd git-2.18.0
make prefix=/usr/local/git all
make prefix=/usr/local/git install
ln -s /usr/local/git/bin/git /bin/git

# 检查
git --version
```

## Node.js 安装

```shell
# 安装依赖
yum install -y epel-release 

# 安装 node
/usr/bin/yum install -y nodejs

# 安装 n<管理版本>
npm install -g n

# 安装最新长期维护版
n lts

# 此时 node -v or npm -v 均还是原来版本
cd /bin/
mv node node_bak
mv npm npm_bak
ln -s /usr/local/bin/node /bin/node
ln -s /usr/local/bin/npm /bin/npm

# 此时 node -v or npm -v 均是最新版本
node -v
npm -v
```

## JDK1.8 安装

```shell
# 下载地址，需要 Oracle 账号
https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html

# 解压下载好的软件包
cd /platform/
tar -zxvf jdk-8u211-linux-x64.tar.gz
 
# 增加环境变量
vim /etc/profile

# 在底部添加
export JAVA_HOME=/platform/jdk1.8.0_211
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH

# 刷新环境变量
source /etc/profile

# 验证
java -version
```

## Maven 安装

```shell
# 此处安装版本为 3.3.9
cd /platform

# 如需换版本，找到对应链接替换该命令即可
wget http://apache.fayea.com/maven/maven-3/3.3.9/binaries/apache-maven-3.3.9-bin.tar.gz

tar -zxvf apache-maven-3.3.9-bin.tar.gz

vim /etc/profile

# 末尾增加
export MAVEN_HOME=/platform/apache-maven-3.3.9
export PATH=${MAVEN_HOME}/bin:${PATH}

# 保存退出
source /etc/profile

# 检查
mvn -v
```

## Nginx 安装

```shell
# 安装 nginx
yum install -y nginx

# 检查并启动
nginx -t && systemctl start nginx 

# 开机自启
systemctl enable nginx.service
```

## MySQL 安装

```shell
# 创建挂载目录
mkdir -p /docker/mysql/conf
mkdir -p /docker/mysql/data

# 修改字符编码
vim /docker/mysql/conf/my.cnf

[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8

# docker 启动容器
docker run -p 13307:3306 \
--name mysql \
-e MYSQL_ROOT_PASSWORD=beluga@mysql. \
--privileged=true \
-v /docker/mysql/data:/var/lib/mysql \
-v /docker/mysql/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

## Jenkins 安装

```shell
#更新yum源
yum update
#下载jenkins
yum install jenkins
#启动jenkins
systemctl start jenkins 
#如果启动失败,则需要重新配置jenkins jdk路径
#查看当前jdk路径，复制路径
which java
#修改jenkins 的jdk路径
vim /etc/init.d/jenkins
#找到后面的java，将/usr/bin/java 修改为自己的jdk路径，然后重启jenkins
systemctl start jenkins
#修改jenkins端口号
sed -i 's/\(JENKINS_PORT=\)"8080"/\1"8888"/' /etc/sysconfig/jenkins
# 需要修改 Jenkins 用户
vim /etc/sysconfig/jenkins

JENKINS_USER="root"
JENKINS_GROUP="root"

#然后重新启动
systemctl restart jenkins

# https://www.cnblogs.com/cnjavahome/p/8975726.html

```

## Redis 安装

```shell
# 带密码加挂载数据
docker run -d \
--name redis \
-p 6379:6379 \
-v /docker/redis/data:/data \
redis --requirepass 'agefades'
```

## MongoDB 安装

```shell
# 创建数据挂载文件目录
mkdir -p /docker/mongo/{conf,data}

# 赋予权限
chmod 777 /docker/mongo/conf
chmod 777 /docker/mongo/data

# 启动容器
docker run --name mongo -d \
-p 27077:27017 \
--privileged=true \
-v /docker/mongo/conf:/data/configdb \
-v /docker/mongo/data:/data/db \
docker.io/mongo:latest \
```

## Wiki 安装

```shell
# https://github.com/phachon/mm-wiki

# 后台启动命令
nohup ./mm-wiki --conf conf/mm-wiki.conf >> /platform/mm-wiki/logs/out.log 2>&1 &
```

## Blog 安装

```shell
# 拉取镜像
docker pull b3log/solo

# 先手动建库（库名 solo，字符集使用 utf8mb4，排序规则 utf8mb4_general_ci

# 启动容器
docker run --detach --name solo --network=host \
--env RUNTIME_DB="MYSQL" \
--env JDBC_USERNAME="root" \
--env JDBC_PASSWORD="beluga@mysql." \
--env JDBC_DRIVER="com.mysql.cj.jdbc.Driver" \
--env JDBC_URL="jdbc:mysql://127.0.0.1:13307/solo?useUnicode=yes&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC" \
--rm \
b3log/solo --listen_port=8080 --server_scheme=http --server_host=www.agefades.com
```

## 禅道安装

```shell
# 拉取镜像
docker pull idoop/zentao  

# 创建挂载目录
mkdir -p /docker/chandao

# 运行容器
docker run -d -p 7777:80 -p 3308:3306 \
        -e USER="root" -e PASSWD="beluga@chandao." \
        -e BIND_ADDRESS="false" \
        -v /docker/chandao:/opt/zbox/ \
        --name chandao \
        idoop/zentao:latest
        
# 浏览器访问页面，初始账号密码为 admin/123456，进入后修改密码
```

## ES6.7 安装

```shell
# 创建挂载目录
mkdir -p /docker/elasticsearch/data

# 赋予权限
chmod -R 777 /docker/elasticsearch/data

# 启动容器
docker run -d \
-e ES_JAVA_OPTS="-Xms3g -Xmx3g" \
-p 19200:9200 \
-p 19300:9300 \
-v /docker/elasticsearch/data:/usr/share/elasticsearch/data \
--name es6.7 docker.io/elasticsearch:6.7.0

# 启动报错
vim /etc/sysctl.conf 
# 最后一行添加
vm.max_map_count=655360
# 保存退出后，执行
sysctl -p
# 重启es
docker start es6.7

# 检测 ES
curl localhost:9200
```

## Kibana 安装

```shell
# 下载镜像
docker pull docker.io/kibana:6.7.0

# 创建挂载目录
mkdir -p /docker/kibana/conf

# 创建配置文件
vim /docker/kibana/conf/kibana.yml

# 编写配置
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://[你自己服务器的ip]:9200" ]
xpack.monitoring.ui.container.elasticsearch.enabled: true

# 启动容器
docker run -d \
--name kibana6.7 \
-p 5601:5601 \
-v /docker/kibana/conf/kibana.yml:/usr/share/kibana/config/kibana.yml \
-e ELASTICSEARCH_URL=http://[你自己ES服务器的ip]:9200 docker.io/kibana:6.7.0 

# 注意端口开放
```

## IK 分词器安装

```shell
# 进入容器内部
docker exec -it  es6.7 bash

# 安装插件
elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.7.0/elasticsearch-analysis-ik-6.7.0.zip

# 下载完成后检查
cd plugins/
ls

# 有 ik 则安装完成
```

## Head 插件安装

```shell
# 拉取镜像
docker pull docker.io/mobz/elasticsearch-head:5

# 启动容器
docker run -d --name head -p 9100:9100 docker.io/mobz/elasticsearch-head:5

# 修改跨域配置
docker cp es6.7:/usr/share/elasticsearch/config/elasticsearch.yml /root/elasticsearch.yml

vim /root/elasticsearch.yml

# 末尾添加
http.cors.enabled: true
http.cors.allow-origin: "*"

docker cp /root/elasticsearch.yml es6.7:/usr/share/elasticsearch/config/elasticsearch.yml

docker restart es6.7
```

## RabbitMQ 安装

```shell
# 拉取镜像
docker pull rabbitmq:3.7.7-management

# 运行容器
docker run -d -p 5671:5617 -p 5672:5672 -p 4369:4369 -p 15671:15671 -p 15672:15672 -p 25672:25672 --name rabbit-3.7.7 rabbitmq:3.7.7-management

# 进入容器
docker exec -it rabbit-3.7.7 /bin/bash

# 进入 plugin 目录
cd plugins

# 更新 apt-get
apt-get update

# 安装下载工具 wget
apt-get install -y wget

# 下载延迟队列插件
wget https://dl.bintray.com/rabbitmq/community-plugins/3.7.x/rabbitmq_delayed_message_exchange/rabbitmq_delayed_message_exchange-20171201-3.7.x.zip

# 安装解压工具 unzip
apt-get install -y unzip

# 解压插件包
unzip rabbitmq_delayed_message_exchange-20171201-3.7.x.zip

# 启动延迟队列插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 修改默认 guest 密码
rabbitmqctl change_password guest swordsman

# 添加用户并授予超级管理员权限
rabbitmqctl add_user swordsman swordsman
rabbitmqctl set_user_tags swordsman administrator

# 赋予 / 虚拟主机权限
rabbitmqctl set_permissions -p / swordsman '.*' '.*' '.*'

# 退出容器
exit

# 重启容器
docker restart rabbit-3.7.7

# 管理台访问地址
[ip]+[port]

# 默认账号密码 guest
```

