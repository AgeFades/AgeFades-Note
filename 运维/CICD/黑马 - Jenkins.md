[TOC]

# 黑马 - Jenkins

## 持续集成

### 简介

- Continuous integration，简称CI
  - 指频繁将代码集成到主干
- 流程：

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638155729029.png)

### 过程

- 根据持续集成的设计，代码从提交到生产，整个过程有如下几步
  - 提交
    - 开发者向仓库提交代码， 所有后续操作，都始于本地代码的一次提交
  - 测试
    - 仓库对commit操作配置钩子(hook)
    - 只要提交代码或者合并到主干，就会跑自动化测试
  - 构建
    - 将源码转换为可以运行的实际代码
    - 比如：安装依赖，配置各种资源（样式表、JS脚本、图片...）
  - 部署
    - 此时代码是一个可以直接部署的版本（artifact）
    - 将该版本的所有文件打包（tar filename.tar*）存档，发布到生产服务器
  - 回滚
    - 一旦当前版本发生问题，就要回滚到上一个版本的构建结果
    - 最简单做法：修改一下符号链接，指向上一个版本的目录

### 组成要素

- 一个自动构建过程，无需人工干预
  - 检出代码、编译构建、运行测试、结果记录、测试统计...
- 一个代码存储库
  - 一般使用 SVN 或 Git
- 一个持续集成服务器
  - 举例：Jenkins

## Jenkins介绍

- [官网](http://jenkins-ci.org/)
- 具有自动化构建、测试、部署等功能

### 特征

- 开源的Java语言开发持续集成工具，支持持续集成，持续部署
- 易于安装部署配置
  - 可通过yum安装、下载war包安装、或docker安装部署
  - 可方便web界面配置管理
- 消息通知及测试报告
  - 集成 RSS/E-mail，通过RSS发布构建结果，或当构建完成时通过 e-mail 通知
  - 生成 JUnit/TestNG 测试报告
- 分布式构建
  - 支持 Jenkins 能够让多台计算机一起构建/测试
- 文件识别
  - jenkins 能跟踪哪次构建生成哪些 jar，哪次构建使用哪个版本的 jar 等
- 丰富的插件支持
  - 支持扩展插件，如 git、maven、docker ...

### 持续集成流程说明

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638157128415.png)

1. 开发提交代码到Git仓库
2. Jenkins使用Git工具，到Git仓库拉取代码
3. 配合JDK、Maven等软件完成代码编译、代码测试、打包
4. 把生成的jar分发到测试服务器、或生产服务器，启动后即可访问应用

## 服务器列表

- 均采用CentOS7

| 名称           | IP地址  | 安装的软件                                          |
| -------------- | ------- | --------------------------------------------------- |
| 代码托管服务器 | 以1代替 | Github-12.4.2                                       |
| 持续集成服务器 | 以2代替 | Jenkins-2.190.3，JDK1.8，Maven3.6.2，Git，SonarQube |
| 应用测试服务器 | 以3代替 | JDK1.8，Tomcat8.5                                   |

## Gitlab安装

- [官网](https://about.gitlab.com/)

- 安装步骤参考 运维/项目服务搭建及部署/component/Gitlab/Gitlab安装.md

## Jenkins安装

- [官方文档](https://pkg.jenkins.io/redhat-stable/)

```bash
# 下载依赖
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo

# 导入秘钥
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# 安装
yum -y install jenkins

# 查找jenkins安装路径
rpm -ql jenkins
```

### Jenkins相关目录

- /user/lib/jenkins/：jenkins安装目录，war包会放在这
- /etc/sysconfig/jenkins：jenkins配置文件，"端口"，”JENKINS_HOME“等都在这配置
- /var/lib/jenkins：默认的 JENKINS_HOME
- /var/log/jenkins/jenkins.log：jenkins日志文件

### 修改配置文件

```bash
vim /etc/sysconfig/jenkins
```

```apl
# 用户名，默认是 jenkins
JENKINS_USER="root"

# 端口号，默认是8080
JENKINS_PORT="8888"
```

### 启动&访问

```bash
systemctl start jenkins
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238354539.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238377338.png)

### 安装插件

#### 修改插件下载地址

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238420356.png)

- 服务器修改配置文件

```bash
cd /var/lib/jenkins/updates

sed -i 's/http:\/\/updates.jenkins-ci.org\/download/https:\/\/mirrors.tuna.tsinghua.edu.cn\/jenkins/g' default.json && sed -i 's/http:\/\/www.google.com/https:\/\/www.baidu.com/g' default.json
```

- 页面更改地址

```apl
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238530188.png)

#### 中文汉化插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238574424.png)

#### 用户权限管理

##### 安装插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238602340.png)

##### 开启权限全局安全配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238688037.png)

##### 创建角色

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238758675.png)

- Global roles：全局角色
- Item roles：针对某个或某些项目的角色，节点相关的权限

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238895381.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638238977140.png)

##### 创建用户

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638239114317.png)

##### 分配角色

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638239332363.png)

##### 创建项目测试权限

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638239360977.png)

### 凭证管理

- 凭证用来存储需要密文保护的数据库密码、Gitlab密码信息、Docker私有仓密码等，以便 Jenkins 可以和这些三方应用交互

#### 安装插件

- Credentials Binding

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638239548720.png)

#### 凭证类型

- Username with password:用户名和密码
-  SSH Username with private key: 使用SSH用户和密钥
-  Secret file:需要保密的文本文件，使用时Jenkins会将文件复制到一个临时目录中，再将文件路径 设置到一个变量中，等构建结束后，所复制的Secret file就会被删除。
-  Secret text:需要保存的一个加密的文本串，如钉钉机器人或Github的api token Certificate:通过上传证书文件的方式

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638239976234.png)

- 常用的是：用户密码、SSH秘钥

#### 安装Git插件和Git工具

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638240082021.png)

```bash
yum -y install git

git --version
```

#### 用户密码类型

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638240739820.png)

##### 测试凭证是否可用

- 修改项目配置、源码管理
- gitlab地址 + 刚创建的凭证、保存即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638241246198.png)

- build一次

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638241292357.png)

#### SSH秘钥类型

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638241622412.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638241639029.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638242101085.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638242066373.png)

## Maven安装和配置

### maven安装&环境变量配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638242440011.png)

### 全局工具配置JDK和Maven

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638244946289.png)

### 添加Jenkins全局变量

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638245038027.png)

### 修改Maven配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638245353988.png)

### 测试Maven是否配置成功

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638245465873.png)

- 保存修改，再次构建测试

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638245476513.png)

## Tomcat安装和配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638251338901.png)

- 注意，tomcat端口默认8080，可能存在端口冲突

### 修改tomcat端口

- 以下操作都在 tomcat 目录下操作

```bash
vim conf/server.xml
```

```xml
<!-- 示例，将8080修改成了8081 -->
<Connector port="8081" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
```

### 查看日志

```bash
less logs/catalina.out
```

### 配置Tomcat用户角色权限

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638251537587.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638251553719.png)

- 后续Jenkins部署项目到Tomcat服务器，需要用到Tomcat的用户
- 修改tomcat配置，添加用户及权限

```bash
vim conf/tomcat-users.xml
```

```xml
<tomcat-users>
    <role rolename="tomcat"/>
    <role rolename="role1"/>
    <role rolename="manager-script"/>
    <role rolename="manager-gui"/>
    <role rolename="manager-status"/>
    <role rolename="admin-gui"/>
    <role rolename="admin-script"/>
    <user username="tomcat" password="tomcat" roles="manager-gui,manager-
script,tomcat,admin-gui,admin-script"/>
</tomcat-users>
```

