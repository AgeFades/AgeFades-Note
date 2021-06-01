# 诸葛 - Nacos - 源码分析

## 源码下载

[源码下载地址](https://github.com/alibaba/nacos/tree/1.4.1)

```shell
git clone https://github.com/alibaba/nacos.git
```

### 切换分支

```shell
# 查看版本标签
git tag

# 切换至 1.4.1
git checkout 1.4.1

# 此时为游离分支，可自己新建分支保存
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622451083790.png)

### 编译源码

```shell
# 在源码目录下执行编译命令
mvn clean install -DskipTests
```

## 主启动类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622444439375.png)

### 单机启动

- 需要加启动参数 -Dnacos.standalone=true

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622451981810.png)

## 服务启动注册流程

### 1. 找到自动装配类

- 根据一般开发规范，中间件的Jar包中，都会使用 spring.factories 对应用注册 bean
- 如下，一般以中间件包前缀命名的 AutoConfiguration，即是核心的自动装配类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622460457468.png)

#### 问题

- 老师演示版本 2.1.0，服务注册核心逻辑是在 NacosDiscoveryAutoConfiguration 类中
- 本人使用版本是 2.2.5，服务注册核心逻辑是在 NacosServiceRegistryAutoConfiguration 里

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622461881899.png)

### 2. 找到自动装配类中基于Spring IOC的扩展点

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462056664.png)

- 看继承抽象类 `AbstractAutoServiceRegistration` 中的实现

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462242994.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462464746.png)

### 3. SpringCloud 中服务注册的顶级接口

- 从上面的 register() 中点下来，可以到 ServiceRegistry 顶级接口
- `ServiceRegistry` 是 SpringCloud 关于服务注册发现的规范顶级接口
  - Nacos 遵循规范，实现了该接口

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462554762.png)

### 4. 注册实例

- Nacos 实现服务注册顶级接口的实现逻辑

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462766893.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462910102.png)

### 5. 发送请求

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622462971092.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622463071450.png)

### 6. Nacos Server的 OpenAPI

[OpenApi地址](https://nacos.io/zh-cn/docs/open-api.html)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622463165635.png)

## Nacos Server 处理服务注册请求

### 1. 接口所在类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622525104303.png)

### 2. 创建或获取服务容器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622525577113.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622525750643.png)

#### 创建服务的逻辑

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622525994027.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622527020259.png)

#### 心跳检测的逻辑

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622526153089.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622526207902.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622526938013.png)

- 客户端心跳检测线程 run() 方法下面还有代码，一屏截不全，简单讲一下
  - 当前时间 - 上次心跳时间 > 实例剔除阈值（30s），就会发送 Http 请求剔除该实例（ deleteIp() ）

