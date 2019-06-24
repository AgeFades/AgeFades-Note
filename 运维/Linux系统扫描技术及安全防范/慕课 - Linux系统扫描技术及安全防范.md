# 慕课 - Linux系统扫描技术及安全防范

## 扫描技术 : 

### ​	一. 主机扫描

#### ​	fping

​		命令 : fping

​		作用 : 批量的给目标主机发送 ping 请求，测试主机的存活情况。

​		特点 : 并行发送、结果易读。

##### 	      fping 安装步骤 : 

​			获取源码包(http://fping.org/)

​			下载命令 : wget http://fping.org/dist/fping-3.10.tar.gz

​			解压并进入目录 : tar -xvf fping-3.10.tar.gz

​						    cd fping-3.10

​			检查配置 : ./configure

​				如有异常 :  configure: error: no acceptable C compiler found in $PATH

​						为主机没有 C 语言环境，安装 gcc 即可。 yum install gcc 

​			编译 : make

​			安装 : make install   （将生成的二进制命令拷贝到系统目录下）

​			检测成功与否 : ls /usr/local/sbin/fping (查看系统命令目录是否存在 fping 命令)

​						fping -v （查看版本）

##### 		fping 参数介绍 :

​			fping -a : 只显示出存活的主机（相反参数 -u）

​			通过标准输入方式 fping + IP1 + IP2

​				-g 支持主机段的方式 fping -g 192.168.1.1 192.168.1.255

​								  fping -g 192.168.1.0/24

​			通过读取一个文件中 IP 内容 fping -f 文件名

#### 	hping 

​		命令 : hping

​		特点 : 支持使用的 TCP/IP 数据包组装、分析工具

##### 	     hping 安装步骤

​			官方站点 : http://www.hping.org/

​			下载 : wget https://github.com/antirez/hping/archive/master.zip

​			解压并进入目录 : unzip master.zip 

​					  	  cd hping-master

​			检查配置 : ./configure

​			编译 : make	

​				如有异常 : 

​					致命错误：pcap.h：没有那个文件或目录

​						缺少系统配置 yum install libpcap-devel	(CentOS)

​								      apt-get install libpcap-devel (ubuntu)

​							     	       rpm -qa | grep pcap-devel (查看系统内文件)

​					net/bpf.h: No such file or directory

​						建立软链接 ln -sf /usr/include/pcap-bpf.h /usr/include/net/bpf.h

​					/usr/bin/ld: cannot find -ltcl

​						yum -y install tcl-devel

​			检查 :  ./hping3 -v   （查看版本）

​			安装 : make install 

​				如有异常 : Can't install the man page: /usr/local/man/man8 does not exist （不用管）

​				hping -h （直接查看命令帮助是否存在）

##### 		hping 常用参数 :

​			对指定目标端口发起 tcp 探测

​				-p 端口 

​				-S 设置 TCP 模式 SYN 包

​			伪造来源 IP ，模拟 Ddos 攻击。

​				-a 仿造 IP 地址

​			netstat -ltn : 列出本机监听的所有 TCP 端口

​			sysctl -w net.ipv4.icmp_echo_ignore_all=1 : 忽略 ICMP 连接方式

​			tcpdump -np -ieth0 : 查看 tcp 通信 ip 和回包 

​			tcpdump -np -ieth0 src host IP地址 : 只查看该 Ip​		

![](<https://i.loli.net/2019/06/18/5d08a8919f74558358.png>)

​		​				

### 	二. 路由扫描

​		作用 : 查询一个主机到另一个主机的经过的路由的跳数、及数据延迟情况

​		常用工具 : traceroute 、mtr

#### 	    mtr : 

​			mtr 特点 : 能测试出主机到每个路由的连通性

#### 	    traceroute : 

##### ​		Tracerouter 原理 : 

​		![Traceroute原理.png](https://i.loli.net/2019/06/18/5d08a96861e5858991.png)

##### ​		traceroute 参数使用 :

​			默认使用的是 UDP 协议（30000 以上的端口）

​			使用 TCP 协议   -T -p 

​			使用 ICMP 协议 -I

​			

### 	三. 批量服务扫描

​		目的 : 批量主机存活扫描。

​			  针对主机服务扫描。

​		作用 : 能更方便快捷获取网络中主机的存活状态。

​			  更加细致、智能获取主机服务侦查情况。

​		典型命令 : nmap 、ncat

#### ​	nmap :

​		![nmap.png](https://i.loli.net/2019/06/18/5d08a98dea72a16898.png)

##### 		安装 : yum install nmap

#### 	     ncat :

​			组合参数 : -w 设置的超时时间 

​					 -z 一个输入输出模式

​					-v 显示命令执行过程

​					-u UDP 扫描

### 	四. Linux 防范恶意扫描安全策略	

#### 		常见的攻击方法 :

​				SYN 攻击

​					利用 TCP 协议缺陷进行，导致系统服务停止响应，网络带宽跑满或者响应缓慢。​					

​				![SYN攻击原理.png](https://i.loli.net/2019/06/18/5d08a9b1168f995086.png)



​				DDOS 攻击

​					分布式访问拒绝服务攻击

​				恶意扫描

##### 		SYN类型DDOS攻击预防

​				减少发送 syn + ack 包时重试次数

​					sysctl -w net.ipv4.tcp_synack_retries=3

​					sysctl -w net.ipv4.tcp_syn_retries=3

​				SYN cookies 技术

​					sysctl -w net.ipv4.tcp_syncookies=1

​				增加 backlog 队列

​					sysctl -w net.ipv4.tcp_max_syn_backlog=2048

##### ​		Linux 下其他预防策略

​				关闭 ICMP 协议请求

​					sysctl -w net.ipv4.icmp_echo_ignore_all=1

​				通过 iptables 防止扫描

​					iptables -A FORWARD -p tcp -syn -m limit --limit 1/s -limit-burst 5 -j ACCEPT

​					iptables -A FORWARD -p tcp -tcp-flags SYN,ACK,FIN,RST RST -m limit --limit 1/s -j ACCEPT

​					iptables -A FORWARD -p icmp -icmp-type echo-request -m limit --limit 1/s -j ACCEPT

​					以上三条命令为教程命令，实际使用中均报错。

​					错误记录 : iptables v1.4.21: unknown option "limit"

## 案例 :	通过网络扫描方式获取某运营商核心设备管理权限

### 	Step1: 

​		通过 tracert 路由跟踪一个公网地址，发现有走内网的核心设备转发。

![tracert.png](https://i.loli.net/2019/06/18/5d08a94903a2595175.png)	

### 	Step2:

​		通过端口范围扫描 nmap ，得知开放了 80(http)、23(Telnet)端口。

​		nmap 用作批量主机服务扫描。

### ​	Step3:

​		尝试进行暴力破解，发现登录 http://10.202.4.73 可以用弱口令进行登录(admin/admin)

### ​	Step4:

​		登录到管理界面后，尝试用 nc 命令进行交互式 shell 操作。

​	![nc.png](https://i.loli.net/2019/06/18/5d08a9075955120944.png)

### ​	Step5:

​		可以进行任何操作

### 	好处 :	

​		不会频繁通过界面去登录而留下痕迹

​		登录非常方便

​		不会被侦测设备侦测到

​		利用 nc 的方式在入侵的机器上给自己留一个后门 

## 总结 :

### ​	网络入侵方式

​		踩点 - 网络扫描 - 查点 - 提权 等