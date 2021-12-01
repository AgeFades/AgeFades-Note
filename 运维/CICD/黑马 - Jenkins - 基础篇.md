[TOC]

# 黑马 - Jenkins - 基础篇

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

#### 修改配置

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
    <user username="tomcat" password="tomcat" roles="manager-gui,manager-script,tomcat,admin-gui,admin-script"/>
</tomcat-users>
```

- 用户和密码都是tomcat

```bash
vim webapps/manager/META-INF/context.xml
```

```xml
<!--
 <Valve className="org.apache.catalina.valves.RemoteAddrValve"
-->
```

- 把上面这行注释掉即可

#### 重启Tomcat

```bash
sh bin/shutdown.sh
sh bin/startup.sh
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638255256894.png)

## Jenkins构建Maven项目

### 构建项目类型介绍

- Jenkins中自动构建项目的类型有很多，常用的有以下三种：
  - 自由风格软件项目（FreeStyle Project）
  - Maven项目（Maven Project）
  - 流水线项目（Pipeline Project）

### 自由风格项目构建

- 下面演示一个自由风格项目来完成项目的集成过程
  - 拉取代码 -> 编译 -> 打包 -> 部署

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638255478834.png)

#### 安装Deploy to container插件

- Jenkins本身无法实现远程部署到Tomcat的功能，需要安装Deploy to container插件实现

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638256102340.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638256287547.png)

#### 演示改动代码后的持续集成

- 修改源码、git提交
- jenkins重新部署
- 再次访问tomcat

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638260139461.png)

### Maven项目构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638260445793.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638261134983.png)

## Pipeline流水线项目构建

### Pipeline简介

#### 概念

- 一套运行在 Jenkins 上的工作流框架
  - 将原来独立运行与单个或多个节点的任务连接起来
  - 实现单个任务难以完成的复杂流程编排和可视化的工作

#### 优势

- 代码：
  - Pipeline以代码的形式实现，通常被检入源代码控制
  - 团队能够编辑、审查、迭代 其构建部署流程
- 持久：
  - 无论是计划内的还是计划外的服务器重启 ，Pipeline都是可恢复的
- 可停止：
  - Pipeline可接收交互式输入，以确定是否继续执行Pipeline
- 多功能：
  - Pipeline支持现实世界中复杂的持续交付要求
  - 支持 fork/join、循环执行、并发执行任务
- 可扩展：
  - Pipeline插件支持其DSL的自定义扩展
  - 以及与其他插件集成的多个选项

#### 如何创建Jenkins Pipeline

- Pipeline脚本是由 Groovy 语言实现的
- Pipeline支持两种语法：
  - Declarative：声明式
  - Scripted Pipeline：脚本式
- Pipeline有两种创建方式：
  - 可以直接在Jenkins的Web UI界面输入脚本
  - 可以通过创建一个Jenkinsfile脚本文件，放入项目源码库中（推荐）

##### 安装Pipeline插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638262246332.png)

### Pipeline语法快速入门

#### Declarerative声明式-Pipeline

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638262530841.png)

##### stages

- 代表整个流水线的所有执行阶段
- 通常stages只有一个，里面包含多个stage

##### stage

- 代表流水线中的某个阶段，可能出现n个
- 一般分为 拉取代码、编译构建、部署 等阶段

##### steps

- 代表一个阶段内需要执行的逻辑
- steps里面是 shell 脚本，git拉取代码，ssh远程发布等任意内容

##### 声明式pipeline示例

```javascript
pipeline {
    agent any

    stages {
        stage('拉取代码') {
            steps {
                echo '拉取代码'
            }
        }
        stage('编译构建') {
            steps {
                echo '编译构建'
            }
        }
        stage('项目部署') {
            steps {
                echo '项目部署'
            }
        }
    }
}

```

##### 遇到问题

- 构建提示，需要重启

```apl
This Pipeline has run successfully, but does not define any stages. Please use the stage step to define some stages in this Pipeline.
```

- 执行重启后，jenkins卡死，查看日志

```bash
less /var/log/jenkins/jenkins.log
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638264295043.png)

- 命令关闭并重启

```bash
systemctl stop jenkins
systemctl start jenkins
```

- 查看Jenkins状态，启动失败

```bash
systemctl status jenkins
```

```apl
# 报错日志
Starting Jenkins File "/usr/bin/java" is not executable
```

- 报错找不到jdk，修改文件

```bash
vim /etc/init.d/jenkins
```

```apl
candidates="
/root/jdk/jdk1.8.0_171/bin/java # 这里是加入的JDK目录
/etc/alternatives/java
/usr/lib/jvm/java-1.8.0/bin/java
/usr/lib/jvm/jre-1.8.0/bin/java
/usr/lib/jvm/java-11.0/bin/java
/usr/lib/jvm/jre-11.0/bin/java
/usr/lib/jvm/java-11-openjdk-amd64
/usr/bin/java
"
```

```bash
vim /etc/sysconfig/jenkins
```

```apl
# 本来是 JENKINS_JAVA_CMD=""，修改成下面这样
JENKINS_JAVA_CMD="$candidate"  
```

- 重新执行

```bash
systemctl daemon-reload
systemctl start jenkins
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638264771737.png)

#### Scripted Pipeline脚本式-Pipeline

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638264884196.png)

##### Node

- 节点，一个Node就是一个Jenkins节点，Master或Agent
  - 是执行Step的具体运行环境
  - 后续讲到Jenkins的Master-Slave架构的时候用到

##### Stage

- 阶段，一个Pipeline可以划分成若干个Stage
  - 每个Stage代表一组操作
  - 比如：Build、Test、Deploy
  - Stage是一个逻辑分组的概念

##### Step

- 步骤，最基本的操作单元，可以是打印一句话、构建一个Docker镜像...
- 由各类Jenkins插件提供，比如命令：sh 'make'，就相当于在终端执行make命令一样

##### 脚本式Pipeline示例

```javascript
node {
    def mvnHome
    stage('拉取代码') { // for display purposes
       echo '拉取代码'
    }
    stage('编译构建') {
       echo '编译构建'
    }
    stage('项目部署') {
       echo '项目部署'
    }
}

```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638266195802.png)

##### Pipeline Script from SCM

- 上面都是在Jenkins UI编写Pipeline，不方便维护
- 建议把 Pipeline 脚本放在项目中，一起进行版本控制

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638323771277.png)

```apl
pipeline {
    agent any

    stages {

        stage('拉取代码') {
            steps {
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: '*/master']],
                        doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                        userRemoteConfigs: [
                            [credentialsId: 'b482f626-2f66-4436-a882-9c89cdac7895',
                             url: 'http://gitlab.agefades.com/test/web_demo.git']
                        ]
                    ]
                )
            }
        }

        stage('编译构建') {
            steps {
                sh label: '', script: 'mvn clean package'
            }
        }

        stage('项目部署') {
            steps {
               deploy adapters: [
                    tomcat8(
                        credentialsId: '5052ff60-5daf-41cb-8162-4cff7c6d16f2',
                        path: '',
                        url: 'http://tomcat.agefades.com/'
                    )
               ],
               contextPath: null,
               war: 'target/*.war'
            }
        }

    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325103205.png)

## 常用构建触发器

- Jenkins内置4种构建触发器
  - 触发远程构建
  - 其他工程构建后触发（Build after other projects and build）
  - 定时构建（Build periodically）
  - 轮训SCM（Poll SCM）

### 触发远程构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325363406.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325466025.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325489501.png)

### 其他工程构建后触发

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325590258.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325862391.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325916418.png)

### 定时构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638325947108.png)

- 定时字符串从左往右分为别：
  - 分、时、日、月、周
- 一些定时表达式的例子：
  - H代表形参
  - 每30分钟构建一次：H/30 * * * *
  - 每2小时构建一次：H H/2 * * *
  - 每天的8点、12点、22点，一天构建3次（多个时间点中间用逗号隔开）
    - 0 8,12,22 * * *
  - 每天中午12点：H 12 * * *
  - 每天下午18点：H 18 * * *
  - 在每个小时的前半个小时内的每10分钟：H(0-29)/10 * * * *
  - 每2小时一次、每个工作日上午9点到下午5点：H H(9-16)/2 * * 1-5

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638327274824.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638327416168.png)

### 轮训SCM

- 定时扫描本地代码仓库的代码是否有变更，如果有就触发项目构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638327136659.png)

### Git hook自动触发构建

- 利用 Gitlab 的 webhook 实现，代码push到仓库、立即触发项目自动构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638327364261.png)

#### 安装Gitlab Hook插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638327397250.png)

#### Jenkins设置自动构建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328071252.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328499722.png)

#### Github配置webhook

##### 1. 开启webhook功能

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328115944.png)

##### 2. 在项目添加webhook

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328144805.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328616984.png)

- 这里配了要拖到页面下方 Add webhook

##### 注意

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638328163628.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329053836.png)

## Jenkins参数化构建

- 项目构建过程中，需要根据用户的输入，动态传入一些参数
  - 从而影响整个构建结果
  - 这时可以使用参数化构建
- Jenkins支持非常丰富的参数类型

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329415543.png)

- 下面举例：通过传参来部署不同分支项目

### 项目创建分支，并推送到Gitlab

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329479246.png)

### 在Jenkins添加字符串类型参数

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329511970.png)

### 改动pipeline脚本

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329549692.png)

### 点击Build with Parameters

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329586714.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329847573.png)

## 配置邮箱服务器发送构建结果

### 安装Email Extension插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638329992836.png)

### Jenkins设置邮箱相关参数

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638330037291.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638330050236.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341491968.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341614410.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341634876.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341685667.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341718494.png)

### 准备邮件内容

- 在项目根目录编写 email.html，并把文件推送到Gitlab

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638337180005.png)

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>${ENV, var="JOB_NAME"}-第${BUILD_NUMBER}次构建日志</title>
</head>

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
      offset="0">
<table width="95%" cellpadding="0" cellspacing="0"
       style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
    <tr>
        <td>(本邮件是程序自动下发的，请勿回复！)</td>
    </tr>
    <tr>
        <td><h2>
            <font color="#0000FF">构建结果 - ${BUILD_STATUS}</font>
        </h2></td>
    </tr>
    <tr>
        <td><br />
            <b><font color="#0B610B">构建信息</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>项目名称&nbsp;：&nbsp;${PROJECT_NAME}</li>
                <li>构建编号&nbsp;：&nbsp;第${BUILD_NUMBER}次构建</li>
                <li>触发原因：&nbsp;${CAUSE}</li>
                <li>构建日志：&nbsp;<a href="${BUILD_URL}console">${BUILD_URL}console</a></li>
                <li>构建&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${BUILD_URL}">${BUILD_URL}</a></li>
                <li>工作目录&nbsp;：&nbsp;<a href="${PROJECT_URL}ws">${PROJECT_URL}ws</a></li>
                <li>项目&nbsp;&nbsp;Url&nbsp;：&nbsp;<a href="${PROJECT_URL}">${PROJECT_URL}</a></li>
            </ul>
        </td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">Changes Since Last
            Successful Build:</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td>
            <ul>
                <li>历史变更记录 : <a href="${PROJECT_URL}changes">${PROJECT_URL}changes</a></li>
            </ul> ${CHANGES_SINCE_LAST_SUCCESS,reverse=true, format="Changes for Build #%n:<br />%c<br />",showPaths=true,changesFormat="<pre>[%a]<br />%m</pre>",pathFormat="&nbsp;&nbsp;&nbsp;&nbsp;%p"}
        </td>
    </tr>
    <tr>
        <td><b>Failed Test Results</b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><pre
                style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">$FAILED_TESTS</pre>
            <br /></td>
    </tr>
    <tr>
        <td><b><font color="#0B610B">构建日志 (最后 100行):</font></b>
            <hr size="2" width="100%" align="center" /></td>
    </tr>
    <tr>
        <td><textarea cols="80" rows="30" readonly="readonly"
                      style="font-family: Courier New">${BUILD_LOG, maxLines=100}</textarea>
        </td>
    </tr>
</table>
</body>
</html>
```

### 编写Jenkinsfile添加构建后发送邮件

```javascript
pipeline {
    agent any

    stages {

        stage('拉取代码') {
            steps {
                echo '拉取代码'
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: '*/${branch}']],
                        doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                        userRemoteConfigs: [
                            [credentialsId: 'b482f626-2f66-4436-a882-9c89cdac7895',
                             url: 'http://gitlab.agefades.com/test/web_demo.git']
                        ]
                    ]
                )
            }
        }

        stage('编译构建') {
            steps {
                echo '编译构建'
                sh label: '', script: 'mvn clean package'
            }
        }

        stage('项目部署') {
            steps {
               deploy adapters: [
                    tomcat8(
                        credentialsId: '5052ff60-5daf-41cb-8162-4cff7c6d16f2',
                        path: '',
                        url: 'http://tomcat.agefades.com/'
                    )
               ],
               contextPath: null,
               war: 'target/*.war'
            }
        }

    }

    post {
        always {
            emailext(
                subject: '构建通知: ${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                body: '${FILE,path="email.html"}',
                to: '18433216@qq.com'
            )
        }
    }

}
```

### 邮件相关全局参数参考列表

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638337541542.png)

### 测试邮件结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638341755019.png)

## Jenkins+SonarQube - 安装

### SonarQube简介

[官网](https://www.sonarqube.org/)

- 一个用于管理代码质量的开放平台，可快速定位代码中潜在的或明显的错误

### 安装SonarQube

#### 安装Mysql+建库

- 注意，这里的MySQL是5.7，8.0版本跟本次实操Sonar版本不匹配

- 需要先安装 MySQL，参考 运维/项目服务搭建及部署/component/mysql

```bash
# 进入mysql容器
docker exec -it mysql mysql -uroot -proot
```

```sql
# 创建sonar数据库
create database sonar;
```

#### 安装sonar

[下载sonar压缩包](https://www.sonarqube.org/downloads/)

- 这里使用的是 SonarQube 6.7.4 版本
  - 本来想用当下最新版9.x的，看了下官方文档只支持jdk11
  - 乱七八糟的配套软件版本太费劲了，就按课程版本来了
- Sonar默认使用9000端口，如果端口被占用，需要修改

```bash
# 没有unzip解压命令的就安装
yum install unzip

# 解压
unzip sonarqube-6.7.4.zip

# 更改位置
mkdir /opt/sonar
mv sonarqube-6.7.4/* /opt/sonar

# 创建sonar用户，因为后面用root用户直接启动会报错、启动失败
useradd sonar
chown -R sonar /opt/sonar
```

##### 修改配置文件

```bash
vim /opt/sonar/conf/sonar.properties
```

```properties
sonar.jdbc.username=root
sonar.jdbc.password=root
sonar.jdbc.url=jdbc:mysql://localhost:3306/sonar?useUnicode=true&characterEncoding=utf8&rewriteBatchedStatements=true&useConfigs=maxPerformance&useSSL=false
```

#### 启动sonar

```bash
cd /opt/sonar

# 启动
su sonar bin/linux-x86-64/sonar.sh start

# 查看状态
su sonar bin/linux-x86-64/sonar.sh status

# 查看日志（logs目录下还有其他软件日志）
less logs/sonar.log
```

##### 问题

[sonar启动问题及解决方案](https://blog.csdn.net/qq_35981283/article/details/81072852)

[mysql8降级到mysql7的问题及解决方案](https://www.jianshu.com/p/7af95b08b954)

- 这里用的是容器部署，所以删的是挂载目录

[sonar JVM Permission denied问题及解决方案](https://blog.csdn.net/jxzdezhanhao/article/details/54376379)

- 这里jdk原定目录在 /root/jdk 下，估计是 sonar 用户权限不足，chown 也给不了权限
- 是将 /root/jdk 下的东西，cp 了一份到 /usr/local/ 下

#### 成功截图

- 已经通过nginx反代域名了

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638345640190.png)

#### 默认账号密码

- admin/admin

#### 创建token

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638345755264.png)

- token要记下来，后面要使用

## Jenkins+Sonar - 实现代码审查

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638345831330.png)

### 安装SonarQube Scanner插件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638346365800.png)

### 添加Sonar凭证

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638346399155.png)

### Jenkins进行Sonar配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638346447213.png)

### Sonar关闭审查结果上传到SCM功能

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638346820855.png)

### 在项目添加Sonar代码审查(Pipeline项目)

#### sonar-project.properties

- 项目根目录下，创建sonar-project.properties文件

```properties
# must be unique in a given SonarQube instance
sonar.projectKey=web_demo_pipline
# this is the name and version displayed in the SonarQube UI. Was mandatory prior to SonarQube 6.1.
sonar.projectName=web_demo_pipline
sonar.projectVersion=1.0

# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
# This property is optional if sonar.modules is set.
sonar.sources=.
sonar.exclusions=**/test/**,**/target/**

sonar.java.source=1.8
sonar.java.target=1.8

# Encoding of the source code. Default is default system encoding
sonar.sourceEncoding=UTF-8
```

#### 修改Jenkinsfile

- 加入sonar代码审查阶段

```groovy
pipeline {
    agent any

    stages {

        stage('拉取代码') {
            steps {
                echo '拉取代码'
                checkout(
                    [
                        $class: 'GitSCM', branches: [[name: '*/${branch}']],
                        doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [],
                        userRemoteConfigs: [
                            [credentialsId: 'b482f626-2f66-4436-a882-9c89cdac7895',
                             url: 'http://gitlab.agefades.com/test/web_demo.git']
                        ]
                    ]
                )
            }
        }

        stage('编译构建') {
            steps {
                echo '编译构建'
                sh label: '', script: 'mvn clean package'
            }
        }

        stage('sonar代码审查') {
            steps {
                script {
                    scannerHome = tool 'sonarqube-scanner'
                }
                withSonarQubeEnv('sonarqube6.7.4') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }

        stage('项目部署') {
            steps {
               deploy adapters: [
                    tomcat8(
                        credentialsId: '5052ff60-5daf-41cb-8162-4cff7c6d16f2',
                        path: '',
                        url: 'http://tomcat.agefades.com/'
                    )
               ],
               contextPath: null,
               war: 'target/*.war'
            }
        }

    }

    post {
        always {
            emailext(
                subject: '构建通知: ${PROJECT_NAME} - Build # ${BUILD_NUMBER} - ${BUILD_STATUS}!',
                body: '${FILE,path="email.html"}',
                to: '18433216@qq.com',
                from: '18433216@qq.com'
            )
        }
    }

}
```

#### 测试结果截图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638348182203.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638348264397.png)

- 刚加入代码审查第一次构建过程会延长很久
- 之后构建经测试，无太大影响

