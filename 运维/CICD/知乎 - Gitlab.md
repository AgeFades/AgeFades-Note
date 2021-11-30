[TOC]

# docker下gitlab安装配置与使用

[原文链接](https://zhuanlan.zhihu.com/p/328795102)

## **一、安装及配置**

### **1. 从`Docker`镜像仓库 拉取`gitlab`镜像**

```text
# gitlab-ce为稳定版本，后面不填写版本则默认pull最新latest版本
# (此步骤时间可能比较久,需要耐心等待⌛️...)
$ docker pull gitlab/gitlab-ce
```

![img](https://pic1.zhimg.com/80/v2-d785454c36eaccc804af4b54072d6cec_1440w.jpg)

如果你需要安装其他版本, **[请去官方镜像库查询其它版本号](https://link.zhihu.com/?target=https%3A//hub.docker.com/r/gitlab/gitlab-ee)** , 或者通过命令查找:

```text
# 从Docker Hub查找镜像
$ docker search gitlab
```

![img](https://pic4.zhimg.com/80/v2-17b5bb7551d5c426191b676e767ad463_1440w.jpg)

参数说明：

- NAME: 镜像仓库源的名称
- DESCRIPTION: 镜像的描述
- OFFICIAL: 是否 docker 官方发布
- stars: 类似 Github 里面的 star，表示点赞、喜欢的意思。
- AUTOMATED: 自动构建。

拉取`gitlab镜像`, 可以通过如下命令查看:

```text
# 列出本地镜像
$ docker images
```

![img](https://pic1.zhimg.com/80/v2-33a7b16547255c8d71153e87ba0e2dc8_1440w.jpg)

### **2.运行gitlab镜像**

首先在运行之前,我们先需要了解一下`docker` 如何`创建`和`运行容器`:

```text
# 创建一个新的容器并运行一个命令
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

通常会将 `GitLab 的配置 (etc) 、 日志 (log) 、数据 (data)`放到容器之外， 便于日后升级， 因此请先准备这三个目录。 在设置其他所有内容之前，请配置一个新的环境变量$GITLAB_HOME ，该变量指向配置，日志和数据文件将驻留的目录。确保目录存在并且已授予适当的权限。

```text
export GITLAB_HOME=$HOME/docker/gitlab
```

$HOME: 指当系统根目录, 需要提前创建好docker/gitlab 目录

```text
# 在系统的根目录执行(存放的路径根据自己随意设置即可)
mkdir docker/gitlab
```

`GitLab`容器使用主机安装的卷来存储持久数据：

| 挂载到宿主机指定目录 | 容器目录 | 说明 |
| -------------------- | -------- | ---- |
|                      |          |      |

```text
sudo docker run --detach \
  --publish 443:443 --publish 80:80 --publish 22:22 \
  --name gitlab \
  --restart always \
  --volume $GITLAB_HOME/config:/etc/gitlab:Z \
  --volume $GITLAB_HOME/logs:/var/log/gitlab:Z \
  --volume $GITLAB_HOME/data:/var/opt/gitlab:Z \
  gitlab/gitlab-ee
```

或者:

```text
$ docker run -d  -p 443:443 -p 9000:80 -p 22:22 --name gitlab --restart always -v $HOME/docker/gitlab/config:/etc/gitlab -v $HOME/docker/gitlab/logs:/var/log/gitlab -v $HOME/docker/gitlab/data:/var/opt/gitlab gitlab/gitlab-ce

# -d：后台运行
# -p：将容器内部端口向外映射
# --name：命名容器名称
# -v：将容器内数据文件夹或者日志、配置等文件夹挂载到宿主机指定目录

# 443:443：将http：443映射到外部端口443
# 9000:80：将web：80映射到外部端口9000
# 22:22：将ssh：22映射到外部端口22
```

运行成功后出现一串字符串

![img](https://pic3.zhimg.com/80/v2-e65cca81fe6830e8d226ca7fc3ace3a6_1440w.jpg)

可以通过如下命令查看已经运行的容器:

```text
# 列出容器, 只显示已经运行的`容器`
docker ps

显示所有的容器，包括未运行的。
docker ps -a 
```

![img](https://pic2.zhimg.com/80/v2-1c6d61131884cabb6bbdcd6a680ba23d_1440w.jpg)

输出详情介绍：

- CONTAINER ID: 容器 ID
- IMAGE: 使用的镜像
- COMMAND: 启动容器时运行的命令
- CREATED: 容器的创建时间
- STATUS: 容器状态
- PORTS: 容器的端口信息和使用的连接类型（tcp\udp）
- NAMES: 自动分配的容器名称

容器的状态有7种：

- created（已创建）
- restarting（重启中）
- running（运行中）
- removing（迁移中）
- paused（暂停）
- exited（停止）
- dead（死亡）

### **3. 配置**

按上面的方式，`gitlab`容器`运行`没问题，但在gitlab上创建项目的时候，生成项目的URL访问地址是按容器的hostname来生成的，也就是容器的id。作为gitlab服务器，我们需要一个固定的URL访问地址，于是需要配置`gitlab.rb` （宿主机路径：$HOME/gitlab/config/gitlab.rb), 配置http协议所使用的访问地址

```text
# 通过vim 来编辑相应的配置, $HOME是当前系统的根目录,根据自己的路径自行修改
vim $HOME/gitlab/config/gitlab.rb

# 配置http协议所使用的访问地址,不加端口号默认为80
external_url 'http://127.0.0.1'

# 配置ssh协议所使用的访问地址和端口
gitlab_rails['gitlab_ssh_host'] = 'http://127.0.0.1'
gitlab_rails['gitlab_shell_ssh_port'] = 22 # 此端口是run时22端口映射的222端口
:wq #保存配置文件并退出
```

修改完之后重启gitlab

```text
# 每次修改gitlab 配置都需要重启
docker restart gitlab
```

### **4.邮箱服务配置**

主要用于`gitlab` 日常使用中邮件通知服务

```text
1.修改配置文件,建议使用企业邮箱
vim $HOME/gitlab/config/gitlab.rb

# 开始邮箱服务
gitlab_rails['smtp_enable'] = true
# 设置邮箱smtp 服务, 根据自己/公司使用的邮箱协议自由设置即可
gitlab_rails['smtp_address'] = "smtp.exmail.qq.com"
# 设置邮箱smtp 服务端口
gitlab_rails['smtp_port'] = 465
# 设置发件人, 建设单独申请邮箱
gitlab_rails['smtp_user_name'] = "getlab@xxx.com"
# 设置登录邮箱密码
gitlab_rails['smtp_password'] = "password"

gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = true
# gitlab发送人, 可以根据自己的需求自己定义
gitlab_rails['gitlab_email_from'] = 'getlab' 
```

使配置生效之后我们可以使用 `gitlab` 自带的工具进行一下测试。依次执行下面的命令：

```text
# 开启 gitlab 的 bash 工具
$ docker exec -it gitlab bash

# 开启 gitlab-rails 工具
$ gitlab-rails console production

# 发送邮件进行测试
Notify.test_email('test@xxx.com', 'Message Subject', 'Message Body').deliver_now
```

## **二、访问我们安装好GitLab**

我们在之前运行的容器时,对外暴露了web 服务的端口是9000, 我们通过这个端口访问我们的安装好`GitLab web客户端`

```text
http://127.0.0.1:9000/
```

在浏览器的地址栏中打开上面地址,访问显示如下图:

![img](https://pic3.zhimg.com/80/v2-f52bc0c9d62528a0bf0d1f129cacc8ca_1440w.jpg)

第一次进入要输入新的`root`用户密码，设置好之后就可以进去`gitlab` 的登录页面

![img](https://pic4.zhimg.com/80/v2-2c31768ff3aaa8967eda21af18fa978f_1440w.jpg)

输入密码用户名`root` 和自己设置的密码, 进入`gitlab`的主界面

![img](https://pic1.zhimg.com/80/v2-67b7c67271ade40c970a54a09041c77c_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-d220d4a2d8ed50729ffa77cda6a6368a_1440w.jpg)

## **三、管理员设置**

管理员主要日常对`gitlab` 的维护, 需要给团队对应的成员分配权限，`Gitlab有管理员角色，拥有很多权限，包括用户的管理，项目的管理，权限的管理`等, 点击导航上的“扳手”(admin Area)图标, 进入管理台控制台:

![img](https://pic1.zhimg.com/80/v2-9152dc3089dd37ac5497c3e84aff4528_1440w.jpg)

### **1.注册限制**

本来一开始是打算让大家自己按照我写好的格式规范注册`GitLab`账号，但是老是有人不遵守规范最后还得我来一个一个的提醒，大大的影响工作效率。因此我决定将`GitLab`的注册功能屏蔽掉，如果有新人进公司需要GitLab账号统一由管理员分配账户给他们。

关于账号出现了以下几个问题:

1. 莫名其妙出现很多陌生人的账号
2. 团队成员的很多账户注册填写的`Email`和`UserName`都不符合规范
3. 不方便管理、`不安全`(离职人员难以控制)

点击工作台,左侧菜单栏“Setting” ---> "General" ---> "Sign-up restrictions"

去掉“Sign-up enabled” & “Require admin approval for new sign-ups” 之后下滑“Save changes”保存即可

![img](https://pic1.zhimg.com/80/v2-ac52216add65395c4d8126b8597917a8_1440w.jpg)

### **2.用户管理**

点击工作台,左侧菜单栏“Users”, 按照页面相应的步骤操作即可,这里就不多描述

![img](https://pic3.zhimg.com/80/v2-608d594320145c8df05ef552dfb2d0ce_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-46572ad6c0d9c21a6476d2deecfb0aa2_1440w.jpg)

主要的说一下权限(Access):

1. Can create group: 是否创建群组
2. Access level:

- Regular: 普通用户, 普通用户可以访问已给权限或者自己的组和项目
- Admin: 管理员, 管理员可以访问所有组、项目和用户、进行`gitlab`的维护

1. External: 外部人员(一般很少用上)

如果用户离职, `gitlab` 用户账号处理有四种:

![img](https://pic1.zhimg.com/80/v2-ef5065be9139097092fc04f4193c1b04_1440w.jpg)

1. Deactivate this user: 停用此用户, 用户将被注销(只有注册用户才会有)
2. Block this user: 保留账号, 将会无法登录、用户将无法访问git存储库、 所有信息都会保留(`建议使用此方式`)
3. Delete user: 删除用户、部分信息会迁移到“Ghost User”
4. Delete user and contributions: 删除用户以及相关贡献

## **四、开始简单的设置GitLab(非管理员)**

### **1. 个人信息设置**

登录成功后, 点击右上角头像 Profile Settings

![img](https://pic1.zhimg.com/80/v2-c9aef984df230445d4062ec4fb771a3c_1440w.jpg)

修改相应的信息之后,滚动页面底部,点击“Update profile settings” 按钮进行更新

### **2. 修改密码**

为了安全起见, 一般用户第一次登录之后都强制修改密码, 如果之后再去要修改密码,可以通过如下流程:

![img](https://pic2.zhimg.com/80/v2-abaca733dced7a23ef00830f2507707d_1440w.jpg)

### **3. 偏好设置**

可以根据自己喜欢风格来设置`gitlab`, 包括:`Navigation theme(导航主题)、Syntax highlighting theme(代码语法高亮主)、Behavior(个性化)、Localization(语言设置)`

Navigation theme & Navigation theme :

![img](https://pic1.zhimg.com/80/v2-9a0730359dd16d97f0ef402e7dd08898_1440w.jpg)

## **五、 Group(组) 管理**

在`gitlab`中, 整个管理的方式是以`组为最小单位`, 名称空间是作为用户名、组名或子组名使用的唯一名称。 创建组的原因有很多, 通过在相同的名称空间下组织相关的项目并向顶级组中添加成员，以更少的步骤授予对多个项目和多个团队成员的访问权。 通过创建一个组并包括适当的成员，可以更容易地在问题中同时提到所有的团队成员，并合并请求。

- 将相关项目组装在一起
- 授予成员一次访问多个项目的权限
- 组也可以嵌套在子组中。

### **1.创建组**

在顶部菜单中，依次单击“Groups”和“Your Groups”，然后单击绿色按钮“New group”

![img](https://pic1.zhimg.com/80/v2-2c95540b41077d94f3cc7f70fd7b0cac_1440w.jpg)

或者，在顶部菜单中，展开plus符号并选择新建组

![img](https://pic3.zhimg.com/80/v2-464db111a4fd019880bfdecaf61110fa_1440w.jpg)

添加以下信息：

![img](https://pic2.zhimg.com/80/v2-00c3352e7905ca1885e4a0ed46b6bf55_1440w.jpg)

- Private: 内部项目, 小组及其项目只能由成员查看
- Internal: 私人只能由项目成员克隆和查看,
- Public: 公共访问, 允许所有者将项目的可见

![img](https://pic2.zhimg.com/80/v2-c13c06e0a807fe73fe23b49584a0f7b5_1440w.jpg)

### **2.删除组 & 编辑**

点击左侧的`Groups`，然后点击需要修改/删除的组

![img](https://pic3.zhimg.com/80/v2-935fc15b90a3b4d92823b602db22554a_1440w.jpg)

![img](https://pic1.zhimg.com/80/v2-6e25983a1716257e2316dd9d47485344_1440w.jpg)

### **3. 组成员添加**

点击左侧的“Groups”，然后点击当然的组 “Members”

![img](https://pic2.zhimg.com/80/v2-c390f886099ba0c4b513467c050bf599_1440w.jpg)

权限类型介绍:

- Guest: 匿名用户,访客, 创建项目、写留言薄
- Reporter: 报告人
  创建项目、写留言薄、拉项目、下载项目、创建代码片段
- Developer: 开发者,创建项目、写留言薄、拉项目、下载项目、创建代码片段、创建合并请求、创建新分支、推送不受保护的分支、移除不受保护的分支 、创建标签、编写wiki
- Master(已经改为Maintainer): 管理者

创建项目、写留言薄、拉项目、下载项目、创建代码片段、创建合并请求、创建新分支、推送不受保护的分支、移除不受保护的分支 、创建标签、编写wiki、增加团队成员、推送受保护的分支、移除受保的分支、编辑项目、添加部署密钥、配置项目钩子

- Owner: 所有者

创建项目、写留言薄、拉项目、下载项目、创建代码片段、创建合并请求、创建新分支、推送不受保护的分支、移除不受保护的分支、创建标签、编写wiki、增加团队成员、推送受保护的分支、移除受保护的分支、编辑项目、添加部署密钥、配置项目钩子、开关公有模式、将项目转移到另一个名称空间、删除项目

Access expiration date: 过期时间,可以给,某个用户添加一个过期时间,时间到了,该用户的权限将会收回

### **4. 组成员编辑 & 删除**

点击左侧的“Groups”，然后点击当然的组 “Members”, 进入用户列表

![img](https://pic1.zhimg.com/80/v2-c4181faa53236d7ab17a7ca12154031c_1440w.jpg)

### **5.给组添加项目**

工作中交集比较多, 每天开发的代码都在组的项目中, 日常操作包括提代码仓库的管理、分支管理、项目用户维护、`Code Review` 等

进入组的详情,点击“new project”按钮, 进去添加页面,输入项目基本信息保存即可.

New project 主要分为三类: blank project(空白的项目)、create from templte(基于社区开源项目模版创建)、import project (从已有的项目导入, 支持很多渠道)

New project:

![img](https://pic3.zhimg.com/80/v2-532986b6896d7b827d2bc8055f820436_1440w.jpg)

create from templte & import project 自己可以去尝试, 这里就不多说了

![img](https://pic2.zhimg.com/80/v2-d7a3dc08754ae8bc12998ad454a40335_1440w.jpg)

## **六、Code Review**

### **1. 目标和原则**

- 提高代码质量，及早发现潜在缺陷，降低修改/弥补缺陷的成本
- 促进团队内部知识共享，提高团队整体水平
- 评审过程对于评审人员来说，也是一种思路重构的过程，帮助更多的人理解系统
- 是一个传递知识的手段，可以让其它并不熟悉代码的人知道作者的意图- 和想法，从而可以在以后轻松维护代码
- 可以被用来确认自己的设计和实现是一个清楚和简单的
- 鼓励相互学习对方的长处和优点
- 高效迅速完成`Code Review`

### **2. 流程和规则**

采用**[Git Flow](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/roverliang/p/9260285.html)** + Pull Request（PR）模式来做Code Review。

![img](https://pic4.zhimg.com/80/v2-45d293c10678283e766eb47c87b41403_1440w.jpg)

Pull Request（`PR`）的说明:

- 通常我们接到一个新的需求时, 需要通过主干分支拉出一个新的本地分支进行开发
- 任务完成才能提交`PR`
- 严禁一个`PR`里面有多个任务，除非它们是紧密关联的
- 在提交PR时,指定相应`Review` 的人员
- 代码`Review` 通过将会合并主干, `删除fear 分支`

发起`Pull Request`以后，请将`Pull Request`的链接通过`Email`发给代码审核者，以此通知对方及时进行审核。(BUG修复类当日必须完成合并或者拒绝，功能类或者觉得有重大调整需要会议Review必须在邮件中明确时间和会议人员)

任务完成时提交`PR`

![img](https://pic3.zhimg.com/80/v2-34ea181573fca52466b178d3f4c104b2_1440w.jpg)

![img](https://pic2.zhimg.com/80/v2-49a8dbc264d046a6c77ecff47ce26879_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-bbdfe4e26331a530e6344c38a987f686_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-90ebcc029632604bc67b599a8e14bd6a_1440w.jpg)

```
PR`提交之后,相应的人员会收到邮箱提醒,处理`PR` 的`Review
```

![img](https://pic4.zhimg.com/80/v2-cbca2022de321667a8087a83ab176c4f_1440w.jpg)

此时,如果发现代码不合规,有`bug` 可以鼠标移动到当前行,进行评价

![img](https://pic1.zhimg.com/80/v2-f1b3879be07d45b2b73ae8cdcdecd788_1440w.jpg)

如果么有问题,将打回进行处理,没有问题进行`merge`即可, 此过程恶心循环, 直到`code` 满意位置

## **七、gitlab 中的权限说明表**

![img](https://pic3.zhimg.com/80/v2-653007aadc518a001f1f2874ff10a5e6_1440w.jpg)

```
注意: 在旧版中Master 等价于新版的 Maintainer
```

## **gitlab-ctl 常用命令**

```text
gitlab-ctl start #启动全部服务
gitlab-ctl restart#重启全部服务
gitlab-ctl stop #停止全部服务
gitlab-ctl restart nginx #重启单个服务，如重启nginx
gitlab-ctl status #查看服务状态
gitlab-ctl reconfigure #使配置文件生效
gitlab-ctl show-config #验证配置文件
gitlab-ctl uninstall #删除gitlab（保留数据）
gitlab-ctl cleanse #删除所有数据，从新开始
gitlab-ctl tail <service name>查看服务的日志
gitlab-ctl tail nginx  #如查看gitlab下nginx日志
gitlab-rails console  #进入控制台
感谢你的耐心阅读, 给你点赞 , 创作不易, 如果对你有帮助,❤️❤️请用你的发财的小手点个赞哦❤️❤️, 谢谢 同时欢迎大家转载, 记得注明出处
```