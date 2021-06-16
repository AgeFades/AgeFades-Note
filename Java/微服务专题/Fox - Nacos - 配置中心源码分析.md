# Fox - Nacos - 配置中心源码分析

## 问题

1. 配置刷新，Client端是如何动态感知的？
2. 多个配置优先级是怎样的？
3. 集群节点是如何同步配置的？

## 架构图

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/F30997AA23404B00A492D8D4D06E92C4/15455)

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/3827B56135FA45879C6FCDB80CAFBA77/14998)

## 入口

- 以 SpringBoot 加载配置的代码逻辑为入口

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623376537551.png)

### 准备加载配置文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623376817709.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623377021139.png)

### 环境配置文件加载

#### PropertySourceLoader

- SpringBoot 提供的加载配置文件的接口

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623377340837.png)

#### 加载顺序

1. bootstrap.yml
2. application.yml
3. application-dev.yml

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623377129648.png)

### 拉取Nacos Server的配置

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623377525414.png)

