# Graylog

## 简介

```shell
# 一个开源的完整的日志管理工具，类似 ELK 技术栈
```

## 官方架构图

![UTOOLS1572487310667.png](https://i.loli.net/2019/10/31/gZPouilKW3Lz9Ax.png)

![UTOOLS1572487339634.png](https://i.loli.net/2019/10/31/zIWPrKN4LpqM9td.png)



```shell
# 其中 MongoDB 用来存储 Graylog 的配置

# ElasticSearch 用来存储日志信息
```

## 为什么需要Graylog

```shell
# 微服务项目中，拆分的细粒度微服务 + 多实例部署，导致日志分布零散，开发人员查询日志困难

# Graylog 将所有日志收集到一起，提供浏览器端日志

# 通过 UDP TCP 等网络连接传输日志数据
```

