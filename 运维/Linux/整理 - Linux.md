# 整理 - Linux

## 基本命令

```shell
clear # 清屏

shutdown -h now # 立即关机

reboot # 重启

cd # 移动目录

ls # 展示所有文件夹以及文件

ll # 以列的方式展示目录

ls -lh # 将文件的大小以 kb 等显示，不会统计文件夹的大小，只会统计文件的大小

ls -la # 查看当前用户根目录下的所有文件

pwd # 显示当前目录

mkdir # 创建目录

date +%Y年%m月%d日' '%H点%M分%S秒   ------》 2018年10月23日 10点35分32秒

date +%F' '%T               ------》 2018-10-23 10:36:52

cal # 显示当前日历

mv # 移动文件、重命名

ls | grep xxx # 模糊查询当前文件夹下含有关键字的文件和文件夹

find xxx # 全局搜索 先手动刷新下系统索引，updatedb

locate xxx # 类似于 find

vim # 创建并编辑文件

cp # 复制文件

cat # 追加、查看、合并文件

du -ach / # 查看目录或者文件的大小

more # 查看文件

less # 查看文件

tail # 从底部开始查看文件

rm # 删除文件

man # 查询命令帮助手册

history # 查询操作日志

ln -s [目标文件或目录名] [新的快捷方式名] # 创建软连接

tar -zxvf # 解压

tar -zcvf # 压缩

unzip # 解压

zip # 压缩

lsblk -f # 查看磁盘分区情况

df -lh # 查看磁盘使用率

service network restart # 重启网卡

ps -aux # 查看所有进程

	PID # 进程号
	%CPU # 进程占用的 CPU 比率
	%MEM # 进程占用的物理内存比例
	VSZ # 进程占用的虚拟内存
	RSS # 进程占用的物理内存
	
ps -ef # 显示进程和父进程ID

kill "进程号" # 杀死进程 -9 参数强制杀死

killall "进程名" # 例 : killall mysql 杀死所有 mysql 进程

systemctl start firewalld.service # 开启防火墙

firewall-cmd --state # 查看防火墙状态

systemctl list-unit-files # 查询所有服务，| grep 加条件

systemctl --type service # 按 q 退出

systemctl disabled "服务名" # 关闭开机自启 enable 开启

netstat -anp | grep "端口号" # 查看端口号占用情况

lsof -i:"端口号" # 检查端口是否开启，被谁开启

useradd "用户名" # 添加用户

passwd "用户名" # 给用户设置密码

whoami # 显示当前用户

userdel "用户名" # 删除用户

cat /etc/passwd # 查询所有用户

groupadd "组名" # 创建用户组

groupdel "组名" # 删除用户组

usermod -g "组名" "用户名" # 移动用户到某用户组

cat /etc/group # 查看用户组信息

ll /home # 显示所有用户

chmod # 赋予权限

chown # 改变文件的所有者

free -h # 显示系统内存使用情况

top # 查看系统 CPU 使用情况
```

## 高级命令

```shell
ps -ef | grep xxx | grep -v grep | awk '{print "kill -9 "$@}' | sh # 杀死 xx 相关进程

echo 3 > /proc/sys/vm/drop_caches # 清除系统内存缓存

docker ps -a | grep xxx | grep -v grep | awk '{print "docker rm "$1}' | sh # 批量删除容器

vmstat 1 # 查看主机运行情况

crontab -l # 查看系统运行的定时任务

# 编写定时任务，清除 docker none 镜像
vim /root/rm-none-image-crontab.sh

# 脚本内容
#!/bin/bash
source /etc/profile
docker images| grep none | grep -v grep| awk '{print "docker rmi "$3}'|sh

# 添加定时任务
crontab -e
SHELL=/bin/bash
*/10 * * * * /bin/bash /root/rm-none-image-crontab.sh # 每10分钟运行一次该脚本

service crond retart # 重启定时任务

systemctl enable crond.service # 设置开机自启
```

