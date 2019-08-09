# 黑马 - K8S

## 官网

<https://kubernetes.io/zh/>

## 快速入门

### 	环境准备

```shell
# 1. 关闭 CentOS 防火墙
systemctl disable firewalld
systemctl stop firewalld

# 2. 安装 etcd 和 kubernetes 软件
yum update
yum install -y etcd kubernetes

# 3. 启动服务
systemctl start etcd
systemctl start docker # 如果启动失败，vim /etc/sysconfig/selinux 把 selinux 后面的改为 disabled，重启机器重启 docker
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy


```

