[参考链接](https://blog.csdn.net/weixin_40533355/article/details/81135112)

## 操作案例

```shell
# 切换超级用户
sudo su

# 下面的命令挨个执行就行
apt remove cmdtest
apt remove yarn
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
apt update
apt install yarn
```

### 安装完成后

```shell
# 在需要进行下载依赖的 node 项目根目录下执行
yarn
```

