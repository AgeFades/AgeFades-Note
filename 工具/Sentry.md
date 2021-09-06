# Sentry

## 资料

[Sentry官网](https://sentry.io/onboarding/cheche/welcome/)

[Sentry集成SpringBoot应用](https://sentry.io/onboarding/cheche/get-started/)

## 简介

- 系统异常、错误日志实时邮件报警的Web服务

## 安装

[Mac本地安装参考资料](https://zhuanlan.zhihu.com/p/405059653)

[Sentry安装及邮件测试](https://zhuanlan.zhihu.com/p/293863914)

- 此文档演示Mac本地安装，Centos、Ubuntu安装大同小异
  - 基于 git clone 源码，install.sh 安装依赖，docker-compose 容器化部署

```shell
# 1.拉取github上sentry的docker配置文件
git clone https://github.com/getsentry/onpremise.git

# 2.执行安装 在onpremise目录下
bash ./install.sh

# 3. 执行 docker-compose up -d
docker-compose up -d
```

- 设置Sentry管理员账号步骤略

## 邮件报警

[Sentry邮件相关问题参考链接](https://juejin.cn/post/6844904051700842510)

- 修改源码目录下的 sentry/config.yml 配置文件
- 示例如下

```yaml
###############
# Mail Server #
###############

mail.backend: 'smtp'  # Use dummy if you want to disable email entirely
mail.host: 'smtp.qq.com'
mail.port: 25
mail.from: '18433216@qq.com'
mail.username: '18433216@qq.com'
mail.password: '****************'
mail.use-tls: false
# mail.use-ssl: false

# NOTE: The following 2 configs (mail.from and mail.list-namespace) are set
#       through SENTRY_MAIL_HOST in sentry.conf.py so remove those first if
#       you want your values in this file to be effective!


# The email address to send on behalf of
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

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897345567.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897364009.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897471961.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630897506209.png)

