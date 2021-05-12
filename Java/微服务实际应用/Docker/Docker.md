## 简介

- Docker网络默认通过端口映射，容器内部 127.0.0.1 不是宿主机的 127.0.0.1
- 此处是方便单机情况，服务组件之间的互访通信

## 创建

- 创建失败的请自信根据错误提示解决问题

```shell
# 创建 docker 网络
docker network create --subnet=173.18.0.0/24 iyobee

# 查询 docker 网络列表
docker network ls
```

