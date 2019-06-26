# 慕课 - Vue + node + mongo

## 导学

![UTOOLS1561426380087.png](https://i.loli.net/2019/06/25/5d1179cf5173352112.png)

![UTOOLS_COMPRESS_1561426415228.png](https://i.loli.net/2019/06/25/5d117a1639f9d69124.png)

![UTOOLS_COMPRESS_1561426510025.png](https://i.loli.net/2019/06/25/5d117a6d30e2978787.png)

![UTOOLS_COMPRESS_1561426612157.png](https://i.loli.net/2019/06/25/5d117acd46c9533722.png)

## 开发环境

### 	微信服务号的申请

### 	云服务器的选购

### 	域名的备案、解析   

​		https://www.dnspod.cn

#### 	node 8以上版本

​		安装 nvm 管理版本，安装 yarn 管理依赖

​	    	 https://yarnpkg.com/zh-Hans/docs/install#centos-stable

​		yarn config set registry https://registry.npm.taobao.org  :  设置淘宝源

​		npm install vue-cli pm2 -g  :  安装脚手架

​		pm2 start | stop | restrat server.js  :  后台形式运行 | 停止 | 重启 Node 服务

​		pm2 list  :  展示 Node 服务

​		pm2 show server  :  更加详细的展示 Node Server

​		pm2 logs  :  查看实时日志  

### 	配置 Nginx 端口代理与域名指向

​		nginx 的负载

```shell
upstream xxx {
    ip_hash;
    server xx.xx.xx.xx:xx;
    server xx.xx.xx.xx:xx;
}
```

### 	安装 Mongo

​		此处本人使用 docker 一键安装

### 	公众号和小程序

​		开通与配置、微信授权，UnionId ... 略

## 微信公众号基础功能快速开发

### 	内网穿透

​		本人用的 uTools

### 	项目初始化

​		vue init nuxt/koa xxx  :  vue 脚手架创建项目 , xxx 为项目名

​		cd xxx | git init | git remote add origin  :  进入目录，初始化 git 并添加远程仓库地址

​		VSCode 打开项目，yarn install  :  安装依赖

​		npm run dev : 启动项目

​			如有异常 ：

```vue
Module build failed: Error: Plugin/Preset files are not allowed to export objects, only functions. In /Users/admin
/Documents/JavaSpace/beluga-node/study/node_modules/backpack-core/babel.js	
```

​			https://www.cnblogs.com/ITtt/p/10515456.html