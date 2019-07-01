# 慕课 - 用Iptables搭建一套强大的安全防护盾

## 关于Iptables :

### ​什么是Iptables?

​	常见于 Linux 系统下的应用层防火墙工具

### 常用人员 : 

​	系统管理人员、网络工程人员、安全人员等。	

### 场景 :

​	模拟用 iptables 控制并发的 http 访问。

![屏幕快照 2019-06-18 下午5.22.29.png](https://i.loli.net/2019/06/18/5d08ad6f17ad880036.png)

​	场景描述 : IP1(通过 ab 命令) -> IP2(http服务)

​	场景操作 : ab -n 10000 -c 40 http://IP地址/资源（例如 http://www,bai.com/a.txt）

​			-n 表示请求总数 

​			-c 表示累增并发数

​			w 命令查看服务器的负载情况

## netfilter 和 iptables

### Netfilter : 

#### 	什么是Netfilter?

​		NetFilter 是 Linux 操作系统核心层内部的一个数据包处理模块。

#### 	什么是Hook point?

​		数据包在 Netfilter 中的挂载点。

​		（PRE_ROUTING、INPUT、OUTPUT、FORWARD、POST_ROUTING）

### Netfilter 和 iptables 的关系

![UTOOLS1560850769174.png](https://i.loli.net/2019/06/18/5d08b1529c07950685.png)

## iptables 规则组成

### 组成部分 :

​	四张表 + 五条链(Hook point) + 规则

#### 	四张表 : 

​		filter 表、nat 表、mangle 表、raw表

​		filter 表 : 访问控制、规则匹配

​		nat 表 : 地址转发

### 数据包在4张表和5条链的流向 :

​	![UTOOLS1560851169125.png](https://i.loli.net/2019/06/18/5d08b2e2ca91413192.png)

### iptables 规则组成 :

#### ​	数据包访问权限 : 

​		ACCEPT : 允许

​		DROP : 丢弃

​		REJECT : 拒绝

#### 	数据包改写 : 

​		SNAT : 对发起端的 Ip 的数据包地址进行改写

​		DNAT : 对目标 Ip 的数据包地址进行改写

#### 	信息记录 : 

​		LOG : 把访问情况记录成 log 形式

​	![UTOOLS1560851759117.png](https://i.loli.net/2019/06/18/5d08b5305898657345.png)

## Iptables 配置

### 场景一 :

#### 	规则一 : 

​		对所有的地址开放本机的 tcp(80、22、10-21)端口的访问

#### ​	规则二 :

​		允许对所有的地址开放本机的基于ICMP协议的数据包访问

#### 	规则三 : 

​		其他未被允许的端口则禁止访问

​	netstat -luntp 命令 : 查看本机服务端口开放情况

​	telnet 目标ip 端口 : 检测能否连接 （例如 telnet 127.0.0.1 22）

​	iptables -nL 命令 : 隐藏主机名，显示当前 iptables 已设置的规则

​	iptables -F 命令 : 清楚之前 iptables 所设置规则

​	iptables -I INPUT -p tcp --dport 80 -j ACCEPT 命令 : （iptables -D INPUT -p tcp --dport 80 -j ACCEPT）

​	iptables -I INPUT -p tcp --dport 22 -j ACCEPT 命令 :

​	iptables -I INPUT -p tcp --dport 10:21 -j ACCEPT 命令 :

​	iptables -I INPUT -p icmp  -j ACCEPT 命令 :

​	iptables -A INPUT -j REJECT 命令 : 此条命令表示原有规则追加，其他任何端口，任何协议都是拒绝。

​		-I : 插入一条规则

​		-D : 删除一条规则

​		INPUT : 入口链

​		-p : 指定连接协议类型

​		--dport  : 目标本机端口（10:21 表示该端口段）

​		-j ACCEPT : 表示允许策略

#### 	存在问题 : 

​		本机无法访问本机

​		本机无法访问其他主机

#### 	解决 :

​		iptables -I INPUT -i lo -j ACCEPT	命令 : 允许本机访问本机

​		iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT   命令 : 允许两种主动往外连接的状态放行。

#### 	补充 : 

​		在场景一的基础上，修改只允许某个 Ip 访问本机的 httpd 服务

​		iptables -D INPUT -p tcp --dport 80 -j ACCEPT   命令 : 清除该条 iptables 规则

​		iptables -I INPUT -p tcp -s 某IP --dport 80 -j ACCEPT : 只允许某个 Ip 访问本机 80端口

### 场景二 :

#### 	ftp 主动模式下的 iptables 的规则配置

​	![UTOOLS1560854467714.png](https://i.loli.net/2019/06/18/5d08bfc4be90125875.png)



#### 	ftp 被动模式下的 iptables 的规则配置

![UTOOLS1560854620153.png](https://i.loli.net/2019/06/18/5d08c05deb2d910006.png)

#### ​	iptables 对于 ftp 主动模式

​		ftp 连接的默认模式为 被动模式

​		vsftpd 服务支持主动模式需要注意配置选项

​			vim /etc/vsftpd/vsftpd.conf

​			port_enable=yes

​			connect_from_port_20=YES

​		iptables 需要开启 21 端口的访问权限

​			iptables -I INPUT -p tcp -dport 21 -j ACCEPT

### 场景三 :

#### 	要求一 : 

​		员工在公司内部（10.10.155.0/24,10.10.188.0/24）能访问服务器上的任何服务。

#### 	要求二 : 

​	 	当员工出差，通过 VPN 连接到公司

​		外网（员工）-> 拨号到 -> VPN 服务器 -> 内网 FTP / SAMBA / NFS / SSH

#### 	要求三 :

​		公司有一个门户网站需要允许公网访问

#### 	常见允许外网访问的服务 :

​		网站 www	http		80/tcp

​					https	443/tcp

​		邮件 mail		smtp	25/tcp

​					smtps	465/tcp

​					pop3	110/tcp

​					pop3s	995/tcp

​					imap	143/tcp

#### 	常见不允许外网访问的服务 :

​		文件服务器 : 	NFS		123/udp

​					SAMBA	137,138,139/tcp 445/tcp

​					FTP		20/tcp,21/tcp

​		数据管理 :	SSH		22/tcp

​		数据库     :	MYSQL	3306/tcp

​					ORACLE	1521/tcp

#### 	配置规则基本思路

​	![UTOOLS1560857281279.png](https://i.loli.net/2019/06/18/5d08cac2195dc25783.png)	

#### 规则设置

​	iptables -I INPUT -i lo -j ACCEPT  命令 : 允许本地回环服务访问。

​	iptables -I INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT  命令 : 允许本地访问外部地址

​	iptables -A  INPUT -s 10.10.100.0/24 (此处为需要放行的网段) -j ACCEPT 命令 : 允许某网段访问本机

​	iptables -A INPUT -p tcp --dport 80 -j ACCEPT  命令 : 允许所有请求 tcp 访问本机 80 端口 (nginx)

​	iptables -A INPUT -p tcp --dport 1723 -j ACCEPT  命令 : 允许所有请求 tcp 访问本机 1723 端口 (VPN)

​	iptables -I INPUT -p icmp -j ACCEPT  命令 : 允许所有请求 icmp 访问本机（icmp 是通信协议，如 ping 某ip）

​	iptables -A INPUT -j REJECT  命令 : 拒绝其他一切没有经过允许的连接。

#### 配置保存

##### ​第一种方式 :	

​	/etc/init.d/iptables save  命令 : 保存 iptables 设置规则

​	vim /etc/sysconfig/iptables  命令 : 查看/编辑 iptables 规则文件

​	chkconfig iptables on   命令 : 设置 iptables 开机自启

​	chkconfig --list | grep iptables : 查看 iptables 在 Linux 各个运行级别的启动情况

##### 第二种方式 :

​	vim /opt/iptable_ssh.sh	命令 : 编写一个 shell 文件，将上述规则置入。

​		#!/bin/sh

​		...

​	vim /etc/rc.loacl  命令 : 编辑开机启动脚本

​		/bin/sh /opt/iptable_ssh.sh 	: 加入后开机自启执行 iptables 规则设置。

## Iptables 防火墙 nat 表设置规则​	

![UTOOLS1560924335651.png](https://i.loli.net/2019/06/19/5d09d0b4b811f83679.png)、

### Iptables 规则中 SNAT 规则设置

![UTOOLS1560924799370.png](https://i.loli.net/2019/06/19/5d09d280bdb2d17957.png)

​	双网卡机器操作 :

​		vim /etc/systcl.conf   命令 : 编写系统服务配置文件

​		net.ipv4.ip_forward = 1 : 开启 Ip 转发，默认为 0 

​		sysctl -p : 配置全部执行

​		sysctl -a | grep ip_forward : 查看 Ip 转发是否为 1

​		iptables -t nat -A POSTROUTING -s 10.10.100.10/24 (此处为某网段) -j SNAT --to 目标 Ip地址

​			此命令为 Iptables 修改 nat 表规则，SNAT 将请求 Ip 在出口时改为目标网址

​		iptables -t nat -L  命令 : 查看 Iptables 关于 NAT 表的设置规则

​	Client 机器操作 :

​		netstat -rn   命令 : 查看本机路由网关情况

​		vim /etc/sysconfig/network  : 

​			加入 GATEWAY=Client IP地址

### Iptables 规则中 DNAT 规则设置

​	![UTOOLS1560925793416.png](https://i.loli.net/2019/06/19/5d09d6627dc2387512.png)

## Iptables 防止 CC 攻击

### connlimit 模块 : 

​	作用 : 用于限制每一个客户端 Ip 的并发连接数

​	参数 : -connlimit-above n	# 限制并发个数

​	例 : iptables -I INPUT -p tcp -syn -dport 80 -m connlimit --connlimit-above 10 -j REJECT

### Limit 模块 :

​	作用 : 限速，控制流量

​	例 : iptables -A INPUT -m limit --limit 3/hour

​	--limit-burst 默认值为 5

​	iptables -A INPUT -p icmp -m limit --limit 1/m --limit-burst 10 -j ACCEPT 

​		iptables 对 INPUT 链追加规则，协议 icmp 

​		-m limit --limit 1/m  : 限流，每分钟进入一个流量包

​		--limit-burst 10  : 最多 10 个流量包，之后的执行 1/m 配置

​	iptables -A INPUT -p icmp -j DROP 

​		关闭默认的 icmp 规则

## Iptables 完整的规则实例介绍

### iptables 实例脚本 : 

#### 系统化的介绍 iptables 规则配置

```shell
/bin/sh
#iptables rules
#Author(作者)	time(时间)
########## ftp ##########
modprobe ipt_MASQUERADE
modprobe ip_conntrack_ftp
modprobe ip_nat_ftp

########## 清除之前规则 ##########
iptables -F
iptables -t nat -F
iptables -X
iptables -t nat -X

########## 允许本机服务回环访问 与 本机与外部通信访问 ##########
iptables -P INPUT DROP
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i lo -j ACCEPT

########## 常用开放端口 与 允许某网段访问 ##########
iptables -A INPUT -p tcp -m multiport --dports 110,80,25 -j ACCEPT	# 110 : POP3 端口 | 25 : smtp 端口
iptables -A INPUT -p tcp -s 10.10.0.0/24 --dport 139 -j ACCEPT	# 139 : samba 协议端口

########## 允许网卡 udp 协议访问 DNS ##########
iptables -A INPUT -i eth1 -p udp -m multiport --dports 53 -j ACCEPT	# 53 : DNS 端口

########## VPN ##########
iptables -A INPUT -p tcp --dport 1723 -j ACCEPT	
iptables -A INPUT -p gre -j ACCEPT # gre : 虚拟隧道

########## 限速与并发防范 ##########
iptables -A INPUT -s 192.186.0.0/24 -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -i ppp0 -p tcp --syn -m connlimit --connlimit-above 15 -j DROP
# ppp0 : 拨号 ，使用 iptables 的 connlimit 模块来限制同一个 ip 发起的连接个数

########## 封闭 icmp 协议 ##########
iptables -A INPUT -p icmp -j DROP

########## nat 内网转发连接外网 ##########
iptables -t nat -A POSTROUTING -o ppp0 -s 10.10.0.0/24 -j MASQUERADE

########## 防止 SYN 攻击 ##########
iptables -N syn-flood
iptables -A INPUT -p tcp --syn -j syn-flood
iptables -I syn-flood -p tcp -m limit --limit 3/s --limit-burst 6 -j RETURN
iptables -A syn-flood -j REJECT

########## 转发 VPN 拨号通信 ##########
iptables -P FORWARD DROP
iptables -A FORWARD -p tcp -s 10.10.0.0/24 -m multiport --dports 80,110,21,25,1723 -j ACCEPT
iptables -A FORWARD -p udp -s 10.10.0.0/24 --dport 53 -j ACCEPT
iptables -A FORWARD -p gre -s 10.10.0.0/24 -j ACCEPT
iptables -A FORWARD -p icmp -s 10.10.0.0/24 -j ACCEPT

########## 内核之开启转发 | SYN Cookies ##########
sysctl -w net.ipv4.ip_forward=1 &>/dev/null
sysctl -w net.ipv4.tcp_syncookies=1

########## 允许本机访问以及所有权限 ##########
iptables -I INPUT -s 10.10.0.50 -j ACCEPT
iptables -I FORWARD -s 10.10.0.50 -j ACCEPT




```

# 	