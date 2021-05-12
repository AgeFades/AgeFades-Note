# Fox - Nacos

## 资料

**重点  **[Nacos官网](https://nacos.io/zh-cn/docs/what-is-nacos.html)

[Nacos Discovery官方快速开始](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery)

## 知识点

- 注册 instance 是怎么存储的？
  - 临时节点存在内存中
  - 持久节点存在磁盘文件 data/naming/namespaced 命名空间的id
- 配置数据存储在哪里？
  - 内嵌的 derby 或者 开发自己的mysql

## Nacos的搭建

