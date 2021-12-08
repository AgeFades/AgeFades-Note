[TOC]

# Sentry

## 资料

[Sentry官网](https://sentry.io/onboarding/cheche/welcome/)

[Sentry集成SpringBoot应用](https://sentry.io/onboarding/cheche/get-started/)

[Sentry使用+最佳实践](https://www.cnblogs.com/hacker-linner/p/15346836.html)

## 简介

- 系统异常、错误日志实时邮件报警的Web服务

## 安装

[Mac本地安装参考资料](https://zhuanlan.zhihu.com/p/405059653)

[Sentry安装及邮件测试](https://zhuanlan.zhihu.com/p/293863914)

### 问题

#### docker-compose版本问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638176708409.png)

- docker-compose版本过高，降低版本

```bash
rm -rf /usr/local/bin/docker-compose

curl -L https://get.daocloud.io/docker/compose/releases/download/1.28.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
```

#### 安装时创建账号问题

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1638179845223.png)

- 填Y，跳过的话后面就不知道账号密码是啥，搜半天没搜着，麻烦
- 填邮箱和两遍密码

如果run ./install.sh 过程中我们没有创建用户，接下来我们可以run docker-compose run --rm web createuser 创建用户。

### 安装步骤

- 此文档演示Mac本地安装，Centos、Ubuntu安装大同小异
  - 基于 git clone 源码，install.sh 安装依赖，docker-compose 容器化部署

```shell
# 1.拉取github上sentry的docker配置文件
git clone https://github.com/getsentry/onpremise.git

# 切换目录
cd onpremise/

# 2.执行安装 在onpremise目录下
bash ./install.sh

# 3. 执行 docker-compose up -d
docker-compose up -d
```

- 默认访问地址: localhost:9000

## 邮件报警

[Sentry邮件相关问题参考链接](https://juejin.cn/post/6844904051700842510)

- 修改源码目录下的 sentry/config.yml 配置文件
- 示例如下

```yaml
###############
# Mail Server #
###############

mail.backend: 'smtp'  
mail.host: 'smtp.qq.com'
mail.port: 25
mail.from: '18433216@qq.com'
mail.username: '18433216@qq.com'
# smtp密码，不是邮箱密码，不会的自行百度
mail.password: '****************'
mail.use-tls: false
```

## 新建项目

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897160471.png)

## 添加成员

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897205005.png)

## 报警规则

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897270866.png)

## 应用接入

- 参考官方对应语言示例文档集成即可

## 测试案例

- 测试代码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897345567.png)

- 请求结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897364009.png)

- Sentry Issues记录

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897471961.png)

- 邮件发送报警

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897506209.png)

- error日志测试代码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1632650407672.png)

- Sentry Issues

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1632706621486.png)

- 通过上图中的 traceId 找到此次异常请求链

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1632650496387.png)
