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

