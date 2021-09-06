[TOC]

# Fox - Nacos - 注册中心

## 资料

[Nacos官网](https://nacos.io/zh-cn/docs/what-is-nacos.html)

[Nacos Discovery官方文档](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-discovery)

## 知识点

- 注册 instance 是怎么存储的？
  - 临时节点存在内存中
  - 持久节点存在磁盘文件 data/naming/namespaced 命名空间的id
- 配置数据存储在哪里？
  - 内嵌的 derby 或者 开发自己的mysql

## Nacos简介

- 关键特性
  - 服务发现和服务健康监测
  - 动态配置服务
  - 动态DNS服务
  - 服务及其元数据管理

### 主流注册中心

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/8820F1E3168547D6A9380F1EAF6F555C/15660)

### Nacos架构

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/15F9A299DD714BF48BDCCB09DFB8ED94/12902)

- `NamingService`：命名服务
  - 注册中心核心接口
- `ConfigService`：配置服务
  - 配置中心核心接口

## Nacos搭建

[参考链接](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8/%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/nacos/Nacos%E5%AE%89%E8%A3%85.md)

## Nacos监控

[参考链接](https://nacos.io/zh-cn/docs/monitor-guide.html)

## Nacos注册中心

### 注册中心演变及其设计思想

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/3268019817E04AF288A736DA705709F7/12942)

### Nacos注册中心架构

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/F94BCA1509DA4586A850C70F042C7BE2/12912)

### 核心功能

- `服务注册`
  - Nacos Client 通过发送 Http Rest 请求 向 Nacos Server 注册自己服务，提供自身元数据
    - 如: ip、port 等...
  - Nacos Server 接收到注册请求后，将这些元数据维护在一个双层的 内存Map 中
- `服务心跳`
  - 服务注册后，Client 会维护一个定时心跳，用于持续通知 Nacos Server，
  - 说明该服务一直处于可用状态，防止被剔除
  - 默认 5秒 发送一次心跳
- `服务同步`
  - Nacos Server 集群间会互相同步服务实例，用来保证服务消息的一致性
    - leader、raft ...
- `服务发现`
  - 消费者在调用 服务提供者 的服务时，会先发送一个 Http Rest 请求到 Nacos Server，
  - 获取到 服务名 对应注册的 服务集合，并缓存在消费者本地，
  - 同时开启一个定时任务，定时拉取 Nacos Server 中服务端最新的 服务集合信息、更新到本地缓存
- `服务健康检查`
  - Nacos Server 会开启一个定时任务，用来检查 注册服务实例的健康情况，
  - 对于超过 15秒 没有收到 Client心跳 的，会将该实例的 Healthy 属性设置为 false
    - 此时消费者调用时仍可调用
  - 对于超过 30秒 没有收到 Client心跳的，将会剔除该实例，直至该实例重新发送心跳 或 重新注册
    - 此时消费者拉取最新服务实例信息为空，调用时就会 没有发现对应提供服务的实例

### 服务注册表结构

- `namespace` 和 `group` 都是起到隔离作用 
  - group 通常用来不同环境配置文件的隔离

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620872522778.png)

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/E4712A0F85D3454CBC16A2DB86D55463/12959)

#### 集群

- 如以机房区分集群:
  - BJ
  - SH
- 不同集群能够互通，但从性能考虑，尽可能使用同一集群

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620873128773.png)

#### 示例

```yaml
spring:	
	cloud:
    nacos:
      # 服务注册
      discovery:
        server-addr: localhost:8848
        namespace: local
        cluster-name: BJ
```

### 服务领域模型

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/99DF6931211144F69D82889D7F7C2909/12960)

### 服务实例数据

- 持久节点实例
  - 存储于磁盘文件
- 临时节点实例
  - 存储于内存（重启即丢失）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620875797447.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620873363242.png)

![](https://note.youdao.com/yws/public/resource/ff9ab83ebe09e367dc598cc844b5bb13/xmlnote/02B8BA9C611C46BD9CEA1CF1AF645E83/15617)

#### 实例集合接口源码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620874616567.png)

```java
/**
     * Get service full information with instances.
     *
     * @param namespaceId namespace id
     * @param serviceName service name
     * @param agent       agent infor string
     * @param clusters    cluster names
     * @param clientIP    client ip
     * @param udpPort     push udp port
     * @param env         env
     * @param isCheck     is check request
     * @param app         app name
     * @param tid         tenant
     * @param healthyOnly whether only for healthy check
     * @return service full information with instances
     * @throws Exception any error during handle
     */
    public ObjectNode doSrvIpxt(String namespaceId, String serviceName, String agent, String clusters, String clientIP,
            int udpPort, String env, boolean isCheck, String app, String tid, boolean healthyOnly) throws Exception {
        
        ClientInfo clientInfo = new ClientInfo(agent);
        ObjectNode result = JacksonUtils.createEmptyJsonNode();
        Service service = serviceManager.getService(namespaceId, serviceName);
        long cacheMillis = switchDomain.getDefaultCacheMillis();
        
        // now try to enable the push
        try {
            if (udpPort > 0 && pushService.canEnablePush(agent)) {
                
                pushService
                        .addClient(namespaceId, serviceName, clusters, agent, new InetSocketAddress(clientIP, udpPort),
                                pushDataSource, tid, app);
                cacheMillis = switchDomain.getPushCacheMillis(serviceName);
            }
        } catch (Exception e) {
            Loggers.SRV_LOG
                    .error("[NACOS-API] failed to added push client {}, {}:{}", clientInfo, clientIP, udpPort, e);
            cacheMillis = switchDomain.getDefaultCacheMillis();
        }
        
        if (service == null) {
            if (Loggers.SRV_LOG.isDebugEnabled()) {
                Loggers.SRV_LOG.debug("no instance to serve for service: {}", serviceName);
            }
            result.put("name", serviceName);
            result.put("clusters", clusters);
            result.put("cacheMillis", cacheMillis);
            result.replace("hosts", JacksonUtils.createEmptyArrayNode());
            return result;
        }
        
        checkIfDisabled(service);
        
        // --- 获取实例 ---
        List<Instance> srvedIPs;
        
        srvedIPs = service.srvIPs(Arrays.asList(StringUtils.split(clusters, ",")));
      
        // --- 获取实例完成 ---
        
        // filter ips using selector:
        if (service.getSelector() != null && StringUtils.isNotBlank(clientIP)) {
            srvedIPs = service.getSelector().select(clientIP, srvedIPs);
        }
        
        if (CollectionUtils.isEmpty(srvedIPs)) {
            
            if (Loggers.SRV_LOG.isDebugEnabled()) {
                Loggers.SRV_LOG.debug("no instance to serve for service: {}", serviceName);
            }
            
            if (clientInfo.type == ClientInfo.ClientType.JAVA
                    && clientInfo.version.compareTo(VersionUtil.parseVersion("1.0.0")) >= 0) {
                result.put("dom", serviceName);
            } else {
                result.put("dom", NamingUtils.getServiceName(serviceName));
            }
            
            result.put("name", serviceName);
            result.put("cacheMillis", cacheMillis);
            result.put("lastRefTime", System.currentTimeMillis());
            result.put("checksum", service.getChecksum());
            result.put("useSpecifiedURL", false);
            result.put("clusters", clusters);
            result.put("env", env);
            result.set("hosts", JacksonUtils.createEmptyArrayNode());
            result.set("metadata", JacksonUtils.transferToJsonNode(service.getMetadata()));
            return result;
        }
        
        Map<Boolean, List<Instance>> ipMap = new HashMap<>(2);
        ipMap.put(Boolean.TRUE, new ArrayList<>());
        ipMap.put(Boolean.FALSE, new ArrayList<>());
        
        for (Instance ip : srvedIPs) {
            ipMap.get(ip.isHealthy()).add(ip);
        }
        
        if (isCheck) {
            result.put("reachProtectThreshold", false);
        }
        
        double threshold = service.getProtectThreshold();
        
        if ((float) ipMap.get(Boolean.TRUE).size() / srvedIPs.size() <= threshold) {
            
            Loggers.SRV_LOG.warn("protect threshold reached, return all ips, service: {}", serviceName);
            if (isCheck) {
                result.put("reachProtectThreshold", true);
            }
            
            ipMap.get(Boolean.TRUE).addAll(ipMap.get(Boolean.FALSE));
            ipMap.get(Boolean.FALSE).clear();
        }
        
        if (isCheck) {
            result.put("protectThreshold", service.getProtectThreshold());
            result.put("reachLocalSiteCallThreshold", false);
            
            return JacksonUtils.createEmptyJsonNode();
        }
        
        ArrayNode hosts = JacksonUtils.createEmptyArrayNode();
        
        for (Map.Entry<Boolean, List<Instance>> entry : ipMap.entrySet()) {
            List<Instance> ips = entry.getValue();
            
            if (healthyOnly && !entry.getKey()) {
                continue;
            }
            
            for (Instance instance : ips) {
                
                // remove disabled instance:
                if (!instance.isEnabled()) {
                    continue;
                }
                
                ObjectNode ipObj = JacksonUtils.createEmptyJsonNode();
                
                ipObj.put("ip", instance.getIp());
                ipObj.put("port", instance.getPort());
                // deprecated since nacos 1.0.0:
                ipObj.put("valid", entry.getKey());
                ipObj.put("healthy", entry.getKey());
                ipObj.put("marked", instance.isMarked());
                ipObj.put("instanceId", instance.getInstanceId());
                ipObj.set("metadata", JacksonUtils.transferToJsonNode(instance.getMetadata()));
                ipObj.put("enabled", instance.isEnabled());
                ipObj.put("weight", instance.getWeight());
                ipObj.put("clusterName", instance.getClusterName());
                if (clientInfo.type == ClientInfo.ClientType.JAVA
                        && clientInfo.version.compareTo(VersionUtil.parseVersion("1.0.0")) >= 0) {
                    ipObj.put("serviceName", instance.getServiceName());
                } else {
                    ipObj.put("serviceName", NamingUtils.getServiceName(instance.getServiceName()));
                }
                
                ipObj.put("ephemeral", instance.isEphemeral());
                hosts.add(ipObj);
                
            }
        }
        
        result.replace("hosts", hosts);
        if (clientInfo.type == ClientInfo.ClientType.JAVA
                && clientInfo.version.compareTo(VersionUtil.parseVersion("1.0.0")) >= 0) {
            result.put("dom", serviceName);
        } else {
            result.put("dom", NamingUtils.getServiceName(serviceName));
        }
        result.put("name", serviceName);
        result.put("cacheMillis", cacheMillis);
        result.put("lastRefTime", System.currentTimeMillis());
        result.put("checksum", service.getChecksum());
        result.put("useSpecifiedURL", false);
        result.put("clusters", clusters);
        result.put("env", env);
        result.replace("metadata", JacksonUtils.transferToJsonNode(service.getMetadata()));
        return result;
    }
```

```java
@JsonInclude(Include.NON_NULL)
public class Service extends com.alibaba.nacos.api.naming.pojo.Service implements Record, RecordListener<Instances> {
	/**
     * Get all instance from input clusters.
     *
     * @param clusters cluster names
     * @return all instance from input clusters, if clusters is empty, return all cluster
     */
  public List<Instance> srvIPs(List<String> clusters) {
    if (CollectionUtils.isEmpty(clusters)) {
      clusters = new ArrayList<>();
      clusters.addAll(clusterMap.keySet());
    }
    // 获取所有实例
    return allIPs(clusters);
  }
  
  /**
     * Get all instance from input clusters.
     *
     * @param clusters cluster names
     * @return all instance from input clusters.
     */
    public List<Instance> allIPs(List<String> clusters) {
        List<Instance> result = new ArrayList<>();
        for (String cluster : clusters) {
            Cluster clusterObj = clusterMap.get(cluster);
            if (clusterObj == null) {
                continue;
            }
            
            // 获取集群的所有实例
            result.addAll(clusterObj.allIPs());
        }
        return result;
    }
  
}
```

```java
public class Cluster extends com.alibaba.nacos.api.naming.pojo.Cluster implements Cloneable {
  	/**
     * Get all instances.
     *
     * @return list of instance
     */
    public List<Instance> allIPs() {
        // 一起拉取所有 临时实例 和 持久实例
        // 所以就不存在 AP 和 CP 的切换
        List<Instance> allInstances = new ArrayList<>();
        allInstances.addAll(persistentInstances);
        allInstances.addAll(ephemeralInstances);
        return allInstances;
    }
}
```

#### 实例临时或持久的配置

```yaml
spring:
	cloud:
    nacos:
      # 服务注册
      discovery:
        server-addr: localhost:8848
        namespace: local
        ephemeral: false # 默认 true 为临时实例，早期版本没有该配置
```

#### 实例注册的原理

- Nacos 实现了 SpringCloud 的 ServiceRegistry 扩展接口，进行服务注册
  - ServiceRegistry 是 SpringCloud 服务注册的核心接口及规范
  - Eureka 是通过 @Inject 注解注入服务，并不优雅规范

```java
public class NacosServiceRegistry implements ServiceRegistry<Registration> {

	public void register(Registration registration) {
        if (StringUtils.isEmpty(registration.getServiceId())) {
            log.warn("No service to register for nacos client...");
        } else {
            NamingService namingService = this.namingService();
            String serviceId = registration.getServiceId();
            String group = this.nacosDiscoveryProperties.getGroup();
            
          // 获取实例
          Instance instance = this.getNacosInstanceFromRegistration(registration);

            try {
              	// 注册实例
                namingService.registerInstance(serviceId, group, instance);
                log.info("nacos registry, {} {} {}:{} register finished", new Object[]{group, serviceId, instance.getIp(), instance.getPort()});
            } catch (Exception var7) {
                log.error("nacos registry, {} register failed...{},", new Object[]{serviceId, registration.toString(), var7});
                ReflectionUtils.rethrowRuntimeException(var7);
            }

        }
    }
  
  /**
   * Nacos 获取实例
   */
  private Instance getNacosInstanceFromRegistration(Registration registration) {
        Instance instance = new Instance();
        instance.setIp(registration.getHost());
        instance.setPort(registration.getPort());
        instance.setWeight((double)this.nacosDiscoveryProperties.getWeight());
        instance.setClusterName(this.nacosDiscoveryProperties.getClusterName());
        instance.setEnabled(this.nacosDiscoveryProperties.isInstanceEnabled());
        instance.setMetadata(registration.getMetadata());
        // 1.4.1 新增的是否临时节点配置属性
    		instance.setEphemeral(this.nacosDiscoveryProperties.isEphemeral());
        return instance;
    }
  
}
```



### Nacos服务注册的集成

[参考链接](https://github.com/AgeFades/AgeFades-Note/blob/master/Java/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%AE%9E%E9%99%85%E5%BA%94%E7%94%A8/%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/nacos/Nacos%E6%95%B4%E5%90%88.md)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620823831793.png)

### 服务间调用

#### 第一次

##### 消费端

```java
@Bean
public RestTemplate restTemplate() {
  return new RestTemplate();
}
```

```java
@Autowired
RestTemplate restTemplate;

@ApiOperation(value = "测试服务间调用")
@GetMapping
public Result test(){
  return restTemplate.getForObject("http://iyobee-points/test", Result.class);
}
```

##### 服务端

```java
@ApiOperation(value = "测试服务间调用")
@GetMapping("test")
public Result test(){
  return Result.success("Hello Nacos!");
}
```

##### 结果

```java
I/O error on GET request for "http://iyobee-points/test": Unexpected end of file from server; nested exception is java.net.SocketException: Unexpected end of file from server
```

#### 第二次

##### 消费端

```java
// 就改这一处
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
  return new RestTemplate();
}

// 上面注解组作用等同于
@Autowired
LoadBalancerClient loadBalancer;

@Bean
public RestTemplate restTemplate() {
  RestTemplate restTemplate = new RestTemplate();
  restTemplate.setInterceptors(Collections.singletonList(new LoadBalancerInterceptor(loadBalancer)));
  return restTemplate;
}
```

##### 结果

- 调用成功

#### 原因

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620870725059.png)

- Nacos 含 Ribbon 负载均衡器的依赖
- RestTemplate 发起 Http 请求前，会调用拦截器链上的方法
  - 扩展点：`ClientHttpRequestInterceptor`
  - LoadBalancer 实现了该拦截器接口

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620870916496.png)

##### LoadBalancerInterceptor 的作用

```java
@Override
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
                                    final ClientHttpRequestExecution execution) throws IOException {
  final URI originalUri = request.getURI();
  String serviceName = originalUri.getHost();
  Assert.state(serviceName != null,
               "Request URI does not contain a valid hostname: " + originalUri);
  
  // 将 host 服务名替换成具体的 ip + port
  // 如上: 将 iyobee-points 替换成 localhost:9006
  return this.loadBalancer.execute(serviceName,
                                   this.requestFactory.createRequest(request, body, execution));
}
```

## Nacos是如何实现自动注册的

### @EnableDiscoveryClient

- 在老的版本中，需要在 Application 中显示标记 @EnableDiscoveryClient 标记注册服务与发现服务
- 新的版本不再需要显示标记该注解

### 图示

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620877570860.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620877642491.png)

### 小结

- 在 Spring IOC 容器加载时，通过监听 IOC加载完成 发布事件，实现服务注册
  - Spring扩展点：ApplicationListener # onApplicationEvent
