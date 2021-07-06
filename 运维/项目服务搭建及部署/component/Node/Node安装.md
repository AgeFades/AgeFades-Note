## 安装

```shell
sudo apt install nodejs

sudo apt install npm
```

### 验证

```shell
node -v
npm -v
```

### 升级最新版本

```shell
# 安装 n<管理版本>
npm install -g n

# 安装最新长期维护版
n lts

# 此时 node -v or npm -v 均还是原来版本
cd /bin/
mv node node_bak
mv npm npm_bak
ln -s /usr/local/bin/node /bin/node
ln -s /usr/local/bin/npm /bin/npm

# 此时 node -v or npm -v 均是最新版本
node -v
npm -v
```

## 问题

- npm run build，node-saas 版本问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623394153048.png)

### 解决方案

```shell
# 卸载
npm uninstall --save sass-loader 
# 安装
npm install sass-loader@7.3.1 --save-dev
# 卸载
npm uninstall --save node-sass 
# 安装
npm i node-sass@4.14.1 
```

