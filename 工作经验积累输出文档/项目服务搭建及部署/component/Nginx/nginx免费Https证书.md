[GitHub项目地址](https://github.com/acmesh-official/acme.sh/wiki/%E8%AF%B4%E6%98%8E)

## 操作案例

```shell
# 切换 root 用户，普通用户会有各种权限问题
sudo su

# 安装命令
curl  https://get.acme.sh | sh

# 安装完成之后可能需要重新打开一个新的Session才有acme命令

# 关闭Nginx，释放80端口 （如果80被占用的话）
sudo docker stop nginx

# 给域名生成证书（把 mydomain.com 换成 自己的域名）
# 我这里操作域名是 dev.admin-api.jdy.iyobee.com
acme.sh  --issue -d test.gezizm.com --standalone
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623311142217.png)

```shell
# 将证书挪到 nginx 配置下
cd /root/.acme.sh
mv dev.admin-api.jdy.iyobee.com/ /home/developer/nginx/config/ssl/

/root/.acme.sh/action-api.gezizm.com_ecc

# 使用里面的 fullchain.cer 和 mydomian.com.key
```

