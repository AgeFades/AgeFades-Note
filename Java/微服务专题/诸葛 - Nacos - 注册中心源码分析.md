[TOC]



# 诸葛 - Nacos - 注册中心源码分析

[源码架构图](https://www.processon.com/view/link/60bd8a991e08533a509e3101)

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

## 核心功能源码架构图

![](https://note.youdao.com/yws/public/resource/17c68958637d60582e9c473f69f04aa5/xmlnote/5323031616A443119754E7B357AC478B/108508)

## Nacos Client 服务注册

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

## Nacos Server 服务注册

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

### 3. 添加实例

- 同步添加任务队列，异步执行注册逻辑
  - 缩短注册实例响应时间、达到更高QPS、TPS
- 临时实例信息，都是在内存中操作（Cluster 的 Set 成员属性）
  - 所以异步实行注册，也能达到准实时的服务注册感知（用户启动完项目就能看到实例注册完成的效果）
  - 一般来说，一次性启动几台、或几十台服务是比较正常的操作
  - 而对内存中的阻塞队列来说，一次性消费几十个消息，速度也是很快的

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622528515679.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622529933790.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622530079663.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622530177580.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622531013261.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622531845795.png)

- Notifier 线程是在项目启动的时候，用线程池开始调用执行（这个自己点两下就能看到调用了， 不截图了）
- 线程里是执行死循环，一直从队列中拉取任务进行处理

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622600482129.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622600601510.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622600884333.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622600931340.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622601078940.png)

#### 注册表结构举例

![](https://note.youdao.com/yws/public/resource/17c68958637d60582e9c473f69f04aa5/xmlnote/A347FEB55DAB4CDC810A41591818B8BA/99347)

### Nacos 注册表高性能读写的实现原理

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622604764509.png)

- 正常来说，在对数据写的操作过程中，有读的请求进来，就可能读到脏数据
  - 这里指 Nacos 的注册表结构，可以类比 MySQL 同时对数据读写可能导致的 脏数据问题

#### 读写分离、写时复制

- Nacos 用的是 CopyOnWrite 的思想实现 Nacos注册表高性能读写
  - 优点：提高读写并发
  - 缺点：数据感知性延后（这种场景基本没啥影响）
  - 其实就是注册实例时，操作注册表的操作过程，是对一个临时变量（原注册表中该namespace、group、service中的cluster的赋值）进行操作，所有操作完毕后，再将它赋值给 原注册表
- Nacos 不会同时有多个 CopyOnWrite  去覆盖原来的注册表数据
  - 因为实例注册，实际是异步完成，单线程消费阻塞队列中的任务

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622615243324.png)

## Nacos Client 服务发现

### 1. 触发时机

- 微服务间第一次通过 OpenFeign 客户端通信时，
  - 底层是 Ribbon 负载均衡拦截器将 服务名替换成实际的 ip:port 
  - 而 Ribbon 完成了 Client 端的服务发现，其实就是 Ribbon 调用了 Nacos Server 的服务发现接口，获取到服务名对应的实例列表

### 2. 开始获取服务所有实例列表

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622684292836.png)

### 3. 尝试从本地缓存中获取

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622684891113.png)

### 4. 本地缓存没有，请求Server端

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622684970544.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622685211549.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622685266476.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622685330146.png)

### 5. Nacos Server的 OpenApi

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622683869429.png)

### 6. 开启延时任务、定时拉取最新服务数据

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1622685483703.png)

- 这里面也是一些逻辑操作，主要就是实现定时刷新服务列表，自己点两下就看懂了，懒得截图了

## Nacos Server 服务发现

- 截图太多了，不截图了，其实大致思路都是一样，Http请求交互，简单记录下核心调用链

### 1. 接收请求

- InstanceController # list() 
  - 接收客户端的请求，响应服务实例集合

### 2. 核心链路

1. InstanceController # doSrvlPXT() 
2. InstanceController # 
   1. Service service = serviceManager.getService(namespaceId, serviceName)
   2. 获取服务对象
3. InstanceController #
   1. List<Instance> srvedIPs = service.srvIPs()
4. Service # 
   1. allIps()
5. Cluster # 

```java
 /**
	* 返回的就是注册时写入的实例数据
 	*/
public List<Instance> allIPs() {
  List<Instance> allInstances = new ArrayList<>();
  allInstances.addAll(persistentInstances);
  allInstances.addAll(ephemeralInstances);
  return allInstances;
}
```

## Nacos Client 心跳检查

### 1. 触发时机

- NacosNamingService.java

```java
@Override
public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {
      
        NamingUtils.checkInstanceIsLegal(instance);
      
        String groupedServiceName = NamingUtils.getGroupedName(serviceName, groupName);
      
  			// 如果是临时实例（默认就是临时）
        if (instance.isEphemeral()) {
          	// 构建一个心跳检测对象
            BeatInfo beatInfo = beatReactor.buildBeatInfo(groupedServiceName, instance);
          	// 将上面构建的心跳检测对象扔到逻辑处理集合里
            beatReactor.addBeatInfo(groupedServiceName, beatInfo);
        }
      
        serverProxy.registerService(groupedServiceName, groupName, instance);
}
```

### 2. 心跳检测属性

- BeatReactor.java

```java
/**
	* Build new beat information.
  */
public BeatInfo buildBeatInfo(String groupedServiceName, Instance instance) {
  BeatInfo beatInfo = new BeatInfo();
  beatInfo.setServiceName(groupedServiceName);	// 组名、服务名
  beatInfo.setIp(instance.getIp());							// IP
  beatInfo.setPort(instance.getPort());					// 端口
  beatInfo.setCluster(instance.getClusterName());	// 集群名
  beatInfo.setWeight(instance.getWeight());			// 权重
  beatInfo.setMetadata(instance.getMetadata());	// 元数据
  beatInfo.setScheduled(false);									
  beatInfo.setPeriod(instance.getInstanceHeartBeatInterval()); // 定时任务检测周期，默认5秒
  return beatInfo;
}
```

### 3. 添加定时任务

- BeatReactor.java

```java

    public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
        NAMING_LOGGER.info("[BEAT] adding beat: {} to beat map.", beatInfo);
        String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
        BeatInfo existBeat = null;
        //fix #1733
        if ((existBeat = dom2Beat.remove(key)) != null) {
            existBeat.setStopped(true);
        }
        dom2Beat.put(key, beatInfo);
      
      	// 往线程池里扔一个定时心跳检测任务，默认5秒执行一次
        executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);
        MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
    }
```

### 4. 心跳任务run()

- BeatReactor # BeatTask

```java
@Override
public void run() {
  if (beatInfo.isStopped()) {
    return;
  }
  // 任务执行时间间隔周期，默认5秒
  long nextTime = beatInfo.getPeriod();
  try {
    // 给 Nacos Server 发心跳检测请求
    JsonNode result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
    long interval = result.get("clientBeatInterval").asLong();
    boolean lightBeatEnabled = false;
    if (result.has(CommonParams.LIGHT_BEAT_ENABLED)) {
      lightBeatEnabled = result.get(CommonParams.LIGHT_BEAT_ENABLED).asBoolean();
    }
    BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
    if (interval > 0) {
      nextTime = interval;
    }
    int code = NamingResponseCode.OK;
    if (result.has(CommonParams.CODE)) {
      code = result.get(CommonParams.CODE).asInt();
    }
    // 如果响应该实例在 Server 端没有，就重新注册
    if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
      Instance instance = new Instance();
      instance.setPort(beatInfo.getPort());
      instance.setIp(beatInfo.getIp());
      instance.setWeight(beatInfo.getWeight());
      instance.setMetadata(beatInfo.getMetadata());
      instance.setClusterName(beatInfo.getCluster());
      instance.setServiceName(beatInfo.getServiceName());
      instance.setInstanceId(instance.getInstanceId());
      instance.setEphemeral(true);
      try {
        serverProxy.registerService(beatInfo.getServiceName(),
                                    NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
      } catch (Exception ignore) {
      }
    }
  } catch (NacosException ex) {
    NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                        JacksonUtils.toJson(beatInfo), ex.getErrCode(), ex.getErrMsg());

  }
  executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
}
```

### 5. 发送心跳检测请求

- NamingProxy.java
- 请求的具体路径、入参出参，都可以去 Nacos 官方 OpenApi 自己看

```java
public JsonNode sendBeat(BeatInfo beatInfo, boolean lightBeatEnabled) throws NacosException {

  if (NAMING_LOGGER.isDebugEnabled()) {
    NAMING_LOGGER.debug("[BEAT] {} sending beat to server: {}", namespaceId, beatInfo.toString());
  }
  Map<String, String> params = new HashMap<String, String>(8);
  Map<String, String> bodyMap = new HashMap<String, String>(2);
  if (!lightBeatEnabled) {
    bodyMap.put("beat", JacksonUtils.toJson(beatInfo));
  }
  params.put(CommonParams.NAMESPACE_ID, namespaceId);
  params.put(CommonParams.SERVICE_NAME, beatInfo.getServiceName());
  params.put(CommonParams.CLUSTER_NAME, beatInfo.getCluster());
  params.put("ip", beatInfo.getIp());
  params.put("port", String.valueOf(beatInfo.getPort()));
  
  // 拼完参数、发送请求
  String result = reqApi(UtilAndComs.nacosUrlBase + "/instance/beat", params, bodyMap, HttpMethod.PUT);
  return JacksonUtils.toObj(result);
}
```

## Nacos Server 心跳检查

### 1. 接收请求

- InstanceController # beat()

### 2. 核心链路

1. 如果实例不存在则重新注册
   1. serviceManager.registerInstance（nameSpaceId, serviceName, instance）
2. service.processClientBeat（clientBeat）
3. 立即开启一个任务 ClientBeatProcessor，更新客户端实例的最后心跳时间
   1. instance.setLastBeat（System.currentTimeMillis）

## Nacos集群

[官方搭建文档](https://nacos.io/zh-cn/docs/deployment.html)

- 默认 AP（高可用、分区容错性） 架构集群
  - 没有主从概念，都是点对点，跟 Eureka 一样
  - AP架构中，集群各方面信息可能会有一定时间的误差，但最终都会达成最终一致性
  - 如果对数据强一致性有要求，就可以采用 CP（强一致性、分区容错性）架构

### 集群架构下的心跳检查机制

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623117467351.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623117615764.png)

- 集群机器数，比如 3个节点，可能第2个节点挂了，这种时候心跳任务 hash 取模可能有会问题
- Nacos 的解决方案：
  - 集群状态下的服务状态同步，允许小范围时间内心跳任务的误差

### 集群架构下的服务状态同步

- ServiceManager # ServiceReporter 定时任务
  - 也是实例启动，开启定时任务，通过 Http 请求跟集群实例同步服务状态信息
- 截图直接参考下面 集群架构下 的节点状态同步，有兴趣自己去源码点点看就行

### 集群架构下的服务数据同步

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623118695137.png)

- 自己点进去看下，也是同步放到任务队列里，异步消费，真正给集群其他节点同步服务数据信息

### 集群架构下的节点状态同步

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623118149776.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623118189743.png)![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623118189743.png)

- 拼接参数、发送请求的代码就不截图了，自己进去点两下就看到了，对应的Http API 也可以去官网找

### 集群架构下新节点启动拉取

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623119416270.png)

- 也是开任务，发Http请求向其他节点拉取数据

