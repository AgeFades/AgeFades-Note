[TOC]

# Mac M1安装虚拟机

## 虚拟机软件下载

[Parallels Desktop](https://www.parallels.com/eu/products/desktop/trial/)

[PD-Runner](https://github.com/lihaoyun6/PD-Runner/releases/tag/0.2.8)

## CentOS镜像

Centos7

链接: https://pan.baidu.com/s/1NqgbugIgg88yD6yAukzxWg 提取码: 1fv5 


Centos8

链接: https://pan.baidu.com/s/1M6809M1b0_1ZRec3Blwe3A 提取码: ogwv 

## 安装步骤

- 打开 Parallels Desktop 控制中心，准备新建虚拟机

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637915837546.png)

- 选择下好的 CentOS 镜像

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637915893681.png)

- 设置虚拟机名字、保存位置、勾选在安装前设定

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637915923343.png)

- 设置cpu、内存，这里示例 2c4g

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637915964067.png)

- 设置共享文件夹

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916013750.png)

- 设置完毕后，准备安装

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916037909.png)

- 光标上调至图中位置，回车后一直按 e 键

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916087952.png)

- 选择语言

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916164922.png)

- 设置 root 密码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916198274.png)

- 软件安装基本开发工具

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916226575.png)

- 选择分区设定，点进去后，点完成，退出到这个页面，就可以下一步了

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916252594.png)

- 安装中，等安装完成即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916300208.png)

- 完成后重启系统，等待即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916465192.png)

- 账号密码登陆

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916502489.png)

- 查看虚拟机ip

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916625235.png)

## 固化静态ip

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637919146687.png)

- 取消勾选即可

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637919159242.png)

## ssh连接

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916725556.png)

## 安装vim

```shell
yum install vim
```

## ssh免密连接

```shell
mkdir .ssh
touch .ssh/authorized_keys
vim .ssh/authorized_keys
```

- 将本地的公钥复制到上面的文件中，保存并退出

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637916862693.png)

- 修改ssh配置文件

```shell
vim /etc/ssh/sshd_config
```

```shell
# 允许密钥认证,此三项在原配置中被注释，可以直接添加到文件末尾
RSAAuthentication yes
PubkeyAuthentication yes
StrictModes no
```

- 保存退出，重启ssh服务

```shell
systemctl restart sshd.service
```

- 本地再用 ssh 连接时，无需再输密码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1637917012019.png)

## 关闭防火墙

```shell
# 停止防火墙
systemctl stop firewalld.service
# 禁止防火墙开机自启
systemctl disable firewalld.service
```

