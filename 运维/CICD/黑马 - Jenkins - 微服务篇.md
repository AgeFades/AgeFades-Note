[TOC]

# 黑马 - Jenkins - 微服务篇

## 微服务持续集成(上)

### 流程说明

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638410234291.png)

1. 开发提交代码到Gitlab
2. Jenkins从Gitlab拉取项目源码，编译打包，然后构建成Docker镜像，将镜像上传到Harbor私有仓
3. Jenkins发送SSH远程命令，让生产部署服务器到Harbor私有仓拉取镜像到本地，然后创建容器
4. 最后，用户访问容器

### 服务列表

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638410436965.png)

### 示例微服务源码概述

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638410485973.png)

### 本地部署

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638411185145.png)

### 环境准备-Docker

#### MySQL Dockerfile官方示例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638412032633.png)

```dockerfile
#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM debian:buster-slim

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

RUN apt-get update && apt-get install -y --no-install-recommends gnupg dirmngr && rm -rf /var/lib/apt/lists/*

# add gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.12
RUN set -eux; \
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates wget; \
	rm -rf /var/lib/apt/lists/*; \
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true

RUN mkdir /docker-entrypoint-initdb.d

RUN apt-get update && apt-get install -y --no-install-recommends \
# for MYSQL_RANDOM_ROOT_PASSWORD
		pwgen \
# for mysql_ssl_rsa_setup
		openssl \
# FATAL ERROR: please install the following Perl modules before executing /usr/local/mysql/scripts/mysql_install_db:
# File::Basename
# File::Copy
# Sys::Hostname
# Data::Dumper
		perl \
# install "xz-utils" for .sql.xz docker-entrypoint-initdb.d files
		xz-utils \
	&& rm -rf /var/lib/apt/lists/*

RUN set -ex; \
# gpg: key 5072E1F5: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported
	key='A4A9406876FCBD3C456770C88C718D3B5072E1F5'; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
	gpg --batch --export "$key" > /etc/apt/trusted.gpg.d/mysql.gpg; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME"; \
	apt-key list > /dev/null

ENV MYSQL_MAJOR 8.0
ENV MYSQL_VERSION 8.0.27-1debian10

RUN echo 'deb http://repo.mysql.com/apt/debian/ buster mysql-8.0' > /etc/apt/sources.list.d/mysql.list

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter
RUN { \
		echo mysql-community-server mysql-community-server/data-dir select ''; \
		echo mysql-community-server mysql-community-server/root-pass password ''; \
		echo mysql-community-server mysql-community-server/re-root-pass password ''; \
		echo mysql-community-server mysql-community-server/remove-test-db select false; \
	} | debconf-set-selections \
	&& apt-get update \
	&& apt-get install -y \
		mysql-community-client="${MYSQL_VERSION}" \
		mysql-community-server-core="${MYSQL_VERSION}" \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld \
	&& chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	&& chmod 1777 /var/run/mysqld /var/lib/mysql

VOLUME /var/lib/mysql

# Config files
COPY config/ /etc/mysql/
COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306 33060
CMD ["mysqld"]
```

#### 使用Dockerfile制作微服务镜像

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638412284532.png)

##### Dockerfile文件

```dockerfile
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
EXPOSE 10086
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

##### 构造镜像

```bash
docker build --build-arg JAR_FILE=tensquare_eureka_server-1.0-SNAPSHOT.jar -t eureka:v1 .
```

##### 创建容器

```bash
docker run -d --name=eureka -p 10086:10086 eureka:v1
```

##### 访问容器

```bash
curl localhost:10086
```

### 环境准备-Harbor

#### 简介

- Docker私有仓库

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638412645328.png)

#### 安装

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638412710604.png)

```bash
tar -zxvf harbor-offline-installer-v1.9.2.tgz
mkdir /opt/harbor
mv harbor/* /opt/harbor
cd /opt/harbor
```

##### 修改配置

```bash
vim harbor.yml
```

```bash
# 修改的配置文件内容
hostname: hub.agefades.com
port: 8880
```

##### 安装Harbor

```bash
./prepare
./install.sh
```

###### 问题

- harbor docker-compose 中有容器名为 nginx 的 service
  - 和已有的 nginx 容器名冲突
- vim dokcer-compose.yml，修改容器名即可

##### 启动Harbor

```bash
# 启动 
docker-compose up -d 
# 停止
docker-compose stop  
# 重新启动
docker-compose restart 
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414148020.png)

##### 默认账号密码

- admin
- Harbor12345

#### 在Harbor创建用户和项目

- Harbor项目分为公开和私有
  - 公开：所有用户都可以访问，通常存放公共镜像，默认有一个library公开项目
  - 私有：只有授权用户才可以访问，通常存放项目本身镜像

##### 创建项目

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414289762.png)

##### 创建用户

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414363244.png)

- 密码Ceshi666

##### 给私有项目分配用户

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414437352.png)

| 角色       | 权限说明                                          |
| ---------- | ------------------------------------------------- |
| 访客       | 对于指定项目拥有只读权限                          |
| 开发人员   | 对于指定项目拥有读写权限                          |
| 维护人员   | 对于指定项目拥有读写权限，创建Webhooks            |
| 项目管理员 | 除了读写权限，同时拥有用户管理/镜像扫描等管理权限 |

##### 以新用户登陆Harbor

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414551315.png)

#### 把镜像上传到Harbor

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638414622118.png)

##### 给镜像打标签

```apl
docker tag eureka:v1 hub.agefades.com/tensquare/eureka:v1
```

##### 登陆Harbor

- 注意，这里需要开放harbor端口

```bash
docker login -u agefades -p Ceshi666 hub.agefades.com
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638416953687.png)

##### 推送镜像

```apl
docker push hub.agefades.com/tensquare/eureka:v1
```

###### 问题1

```apl
The push refers to repository [hub.agefades.com/tensquare/eureka]
Get "https://hub.agefades.com/v2/": dial tcp 117.50.175.156:443: connect: connection refused
```

###### 将Harbor地址加入到Dokcer信任列表

```bash
vim /etc/docker/daemon.json
```

```json
{
  "registry-mirrors": ["https://82m9ar63.mirror.aliyuncs.com"]
  "insecure-registries": ["hub.agefades.com"]
}
```

###### 注意

- 这里 hub.agefades.com 需要配置 https 证书
- 参考运维/项目服务搭建及部署/component/Nginx/nginx免费https证书

###### 问题2

- docker推送镜像，上传至经过nginx反向代理的harbor报错unkown blob

[参考资料](https://blog.csdn.net/kingboyworld/article/details/80109916)

#### 另外机器从Harbor拉取

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638425395739.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638426665113.png)

### 项目代码上传到Gitlab

- 导入项目到本地编辑器，修改配置文件、乱七八糟的，让项目成功run起来
- 修改git远端

```bash
# 查看远端列表
git remote -v

# 删除原来的远端
git remote rm [原来的远端名]

# 新增远端地址
git remote add origin [git项目地址]

# 添加本地文件到本地仓
git add ./*

# 提交修改
git commit -m "feat: 修改远端地址"

# 推送至远端
git push --set-upstream origin master
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638426684628.png)

### 从Github拉取项目源码

#### 创建Jenkinsfile文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638427516881.png)

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_back.git']
                ]
            ]
        )
    }

}
```

#### 拉取Jenkinsfile文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638427572744.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638427589578.png)

### 提交到SonarQube代码审查

#### 添加参数设置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638427649651.png)

#### 每个项目根目录下添加sonar-project.properties

- 注意修改 sonar.projectKey、sonar.projectName

```properties
# must be unique in a given SonarQube instance
sonar.projectKey=tensquare_zuul
# this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
sonar.projectName=tensquare_zuul
sonar.projectVersion=1.0

# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set.
sonar.sources=.
sonar.exclusions=**/test/**,**/target/**
sonar.java.binaries=.

sonar.java.source=1.8
sonar.java.target=1.8
#sonar.java.libraries=**/target/classes/**

# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```

#### 修改Jenkinsfile构建脚本

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"
// 构建版本的名称
def tag = "latest"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_back.git']
                ]
            ]
        )
    }

    stage('代码审查') {
        def scannerHome = tool 'sonarqube-scanner'
        withSonarQubeEnv('sonarqube6.7.4') {
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
        }
    }

}
```

#### 操作结果示例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638428008333.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638428026758.png)

### 使用Dockerfile编译、生成镜像

#### pom添加dockerfile插件

```xml
<plugin>
  <groupId>com.spotify</groupId>
  <artifactId>dockerfile-maven-plugin</artifactId>
  <version>1.3.6</version>
  <configuration>
    <repository>${project.artifactId}</repository>
    <buildArgs>
      <JAR_FILE>target/${project.build.finalName}.jar</JAR_FILE>
    </buildArgs>
  </configuration>
</plugin>
```

#### 每个项目根目录下建立Dokcerfile文件

- 注意：每个项目端口不一样

```dockerfile
#FROM java:8
FROM openjdk:8-jdk-alpine
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
EXPOSE 9001
ENTRYPOINT ["java","-jar","/app.jar"]
```

#### 修改Jenkinsfile构建脚本

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"
// 构建版本的名称
def tag = "latest"
// Harbor私服地址
def harbor_url = "hub.agefades.com/tensquare/"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_back.git']
                ]
            ]
        )
    }

    stage('代码审查') {
        def scannerHome = tool 'sonarqube-scanner'
        withSonarQubeEnv('sonarqube6.7.4') {
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
        }
    }

    stage('编译&构建镜像') {
        // 定义镜像名称
        def imageName = "${project_name}:${tag}"
        // 编译、安装公共工程
        sh "mvn -f tensquare_common clean install"
        // 编译、构建本地镜像
        sh "mvn -f ${project_name} clean package dockerfile:build"
    }

}
```

#### 问题1

```apl
No plugin found for prefix 'dockerfile' in the current project and in the plugin groups [org.apache.maven.plugins, org.codehaus.mojo] available from the repositories [local (/root/repo), alimaven 
```

[参考资料](https://cloud.tencent.com/developer/article/1561264)

- 在settings.xml中添加配置

```xml
<pluginGroups>
  <pluginGroup>com.spotify</pluginGroup>
</pluginGroups>
```

#### 问题2

- 如果出现找不到父工程依赖，需要手动把父工程的依赖上传到仓库中

```apl
[ERROR] Failed to execute goal on project tensquare_admin_service: Could not resolve dependencies for project com.tensquare:tensquare_admin_service:jar:1.0-SNAPSHOT: Failed to collect dependencies at com.tensquare:tensquare_common:jar:1.0-SNAPSHOT: Failed to read artifact descriptor for com.tensquare:tensquare_common:jar:1.0-SNAPSHOT: Could not find artifact com.tensquare:tensquare_parent:pom:1.0-SNAPSHOT -> [Help 1]
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638429533193.png)

#### 成功案例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638429709079.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638429742515.png)

### 上传到Harbor镜像仓库

#### 添加Harbor凭证

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638429974778.png)

#### 修改jenkinsfile构建脚本

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"
// 构建版本的名称
def tag = "latest"
// Harbor私服地址
def harbor_url = "hub.agefades.com"
// Harbor项目名称
def harbor_project_name = "tensquare"
// Harbor的凭证
def harbor_auth = "e1e8d5fc-b289-404f-858e-1c4d9f1bbd6b"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_back.git']
                ]
            ]
        )
    }

    stage('代码审查') {
        def scannerHome = tool 'sonarqube-scanner'
        withSonarQubeEnv('sonarqube6.7.4') {
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
        }
    }

    stage('编译&构建镜像') {
        // 定义镜像名称
        def imageName = "${project_name}:${tag}"
        // 编译、安装公共工程
        sh "mvn -f tensquare_common clean install"
        // 编译、构建本地镜像
        sh "mvn -f ${project_name} clean package dockerfile:build"
        // 给镜像打标签
        sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"
        // 登陆Harbor,并上传镜像
        withCredentials(
            [usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]
        ) {
            // 登录
            sh "docker login -u ${username} -p ${password} ${harbor_url}"
            // 上传镜像
            sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"
        }

        // 删除本地镜像
        sh "docker rmi -f ${imageName}"
        sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${imageName}"
    }

}
```

#### 成功案例截图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638431086708.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638431112670.png)

### 拉取镜像和发布应用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638431198481.png)

#### 安装Publish Over SSH插件

##### 问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638431530380.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638431779038.png)

#### Jenkins服务器生成公私钥

```bash
ssh-keygen -m PEM -t rsa -b 4096 -C "18433216@qq.com"
```

#### 配置远程部署服务器

1. 拷贝公钥到远程服务器

```bash
ssh-copy-id 192.168.66.103
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638432022519.png)

2. 系统配置->添加远程服务器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638432064921.png)

##### 问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638433792418.png)

```apl
Exception when publishing, exception message [Failed to add SSH key. Message [invalid privatekey: [B@60361929]]
```

##### 解决

[参考链接](https://www.cnblogs.com/architectforest/p/13707244.html)

#### 修改Jenkinsfile构建脚本

- 生成远端调用模板代码
- 添加一个port参数

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638432238822.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638432469709.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638432490156.png)

- 注意，上面模板脚本中 execCommand: 添加了执行命令

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"
// 构建版本的名称
def tag = "latest"
// Harbor私服地址
def harbor_url = "hub.agefades.com"
// Harbor项目名称
def harbor_project_name = "tensquare"
// Harbor的凭证
def harbor_auth = "e1e8d5fc-b289-404f-858e-1c4d9f1bbd6b"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_back.git']
                ]
            ]
        )
    }

    stage('代码审查') {
        def scannerHome = tool 'sonarqube-scanner'
        withSonarQubeEnv('sonarqube6.7.4') {
            sh """
                cd ${project_name}
                ${scannerHome}/bin/sonar-scanner
            """
        }
    }

    stage('编译&构建镜像') {
        // 定义镜像名称
        def imageName = "${project_name}:${tag}"
        // 编译、安装公共工程
        sh "mvn -f tensquare_common clean install"
        // 编译、构建本地镜像
        sh "mvn -f ${project_name} clean package dockerfile:build"
        // 给镜像打标签
        sh "docker tag ${imageName} ${harbor_url}/${harbor_project_name}/${imageName}"
        // 登陆Harbor,并上传镜像
        withCredentials(
            [usernamePassword(credentialsId: "${harbor_auth}", passwordVariable: 'password', usernameVariable: 'username')]
        ) {
            // 登录
            sh "docker login -u ${username} -p ${password} ${harbor_url}"
            // 上传镜像
            sh "docker push ${harbor_url}/${harbor_project_name}/${imageName}"
        }

        // 删除本地镜像
        sh "docker rmi -f ${imageName}"
        sh "docker rmi -f ${harbor_url}/${harbor_project_name}/${imageName}"

        // =====以下为远程调用进行项目部署=====
      	sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: "/opt/jenkins_shell/deploy.sh $harbor_url $harbor_project_name $project_name $tag $port", execTimeout: 1200000, execInPty: true, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '', remoteDirectorySDF: false, removePrefix: '', sourceFiles: '')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])

    }

}
```

#### 编写deploy.sh部署脚本

```bash
#! /bin/sh
# 接收外部参数
harbor_url=$1
harbor_project_name=$2
project_name=$3
tag=$4
port=$5

imageName=$harbor_url/$harbor_project_name/$project_name:$tag
echo "$imageName"

# 查询容器是否存在，存在则删除
containerId=`docker ps -a | grep -w ${project_name}:${tag} | awk '{print $1}'`
if [ "$containerId" != "" ] ; then
	# 停掉容器&删除容器
	docker stop $containerId
	docker rm $containerId
	echo "成功删除容器"
fi

# 查询镜像是否存在，存在则删除
imageId=`docker images | grep -w $project_name | awk '{print $3}'`
if [ "$imageId" != "" ] ; then
	# 删除镜像
	docker rmi -f $imageId
	echo "成功删除镜像"
fi

# 登陆Harbor私服
docker login -u agefades -p Ceshi666 $harbor_url

# 下载镜像
docker pull $imageName

# 启动容器
docker run -di -p $port:$port $imageName
echo "容器启动成功"
```

- 上传deploy.sh到目标远端部署服务器 /opt/jenkins_shell目录下，且文件至少有执行权限！

```bash
# 添加执行权限
chmod +x deploy.sh
```

##### 问题

- vim编辑中文乱码

[参考链接](https://www.jianshu.com/p/c1b3566bcb4a)

```bash
vim ~/.vimrc
```

```bash
set fileencodings=utf-8,ucs-bom,gb18030,gbk,gb2312,cp936
set termencoding=utf-8
set encoding=utf-8
```

#### 导入数据，测试微服务

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638433200991.png)

#### 问题1

```apl
Exception when publishing, exception message [An SSH Transfer Set must not have an empty Source files and an empty Exec command - the transfer set should transfer files, execute a command or do both]
```

##### 说明

- 生成远端调用模板后，没在里面添加要执行的sh文件目录及各参数

#### 问题2

```apl
ERROR: Exception when publishing, exception message [Exec exit status not zero. Status [127]]
```

##### 说明

- 远端部署要执行的sh脚本，要放在对应的远端服务器对应目录，请检查

#### 问题3

```apl
ERROR: Exception when publishing, exception message [Exec exit status not zero. Status [126]]
```

##### 说明

- 远端将被执行的sh脚本权限不足，需要给该脚本加上可执行权限

#### 问题4

```apl
ERROR: Exception when publishing, exception message [Exec timed out or was interrupted after 120,000 ms]
```

##### 说明

- 在远端服务器执行脚本超时，需要在 sshPublisher 脚本里，延长超时时间设定 

```apl
# 举例
execTimeout: 1200000, execInPty: true
```

### 部署前端静态web网站

#### 安装NodeJS插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638439410739.png)

#### Jenkins配置Node

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638439538684.png)

#### 创建前端流水线项目

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638502308869.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638502321254.png)

#### 建立Jenkinsfile构建脚本

```groovy
// gitlab ssh凭证
def git_auth = "cb866f73-0f65-4654-89f2-f4d2f0e18cbc"

node {

    stage('拉取代码') {
        checkout(
            [
                $class: 'GitSCM', branches: [[name: '*/${branch}']],
                doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                userRemoteConfigs: [
                    [
                        credentialsId: "${git_auth}", url: 'ssh://git@gitlab.agefades.com:10022/test/tensquare_front.git'
                    ]
                ]
            ]
        )
    }

    stage('打包&部署网站') {
        // 使用NodeJS的npm包进行打包
        nodejs('nodejs17') {
            sh '''
                npm install
                npm run build
            '''
        }

        // =====以下为远程调用进行项目部署======
        sshPublisher(publishers: [sshPublisherDesc(configName: 'master_server', transfers: [sshTransfer(cleanRemote: false, excludes: '', execCommand: '', execTimeout: 1200000, flatten: false, makeEmptyDirs: false, noDefaultExcludes: false, patternSeparator: '[, ]+', remoteDirectory: '/data/nginx/data/tensquare_front', remoteDirectorySDF: false, removePrefix: 'dist', sourceFiles: 'dist/**')], usePromotionTimestamp: false, useWorkspaceInPromotion: false, verbose: false)])
    }

}
```

#### 问题

- npm install到处报错，本人是后端开发，先跳过

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638502417976.png)

## 微服务持续集成(下)

### 部署方案优化

- 上面部署方案存在的问题：
  - 一次只能选择一个微服务部署
  - 只有一台生产者部署服务器
  - 每个微服务只有一个实例，容错率低
- 优化方案
  - 在一个Jenkins工程中，可以选择多个微服务同时发布
  - 在一个Jenkins工程中，可以选择多台生产服务器同时部署
  - 每个微服务都是以 `集群高可用` 形式部署

### 集群部署流程说明

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638515284449.png)

