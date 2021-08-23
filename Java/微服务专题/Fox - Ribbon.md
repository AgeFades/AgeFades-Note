[TOC]

# Fox - Ribbon

## 主流负载方案

- 集中式负载均衡
  - 在 **消费者** 和 **服务提供方** 中间使用独立的代理方案进行负载
    - 硬件案例: F5
    - 软件案例: Nginx

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/074F758C9974417991452EC230F43ED5/13572)

- 客户端按需负载
  - 客户端会先拿到需调用服务的实例数据集合，在发送请求前通过负载均衡算法选定某一个发送
  - 即：在客户端就进行负载均衡算法分配
    - 案例: Ribbon

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/BD4B711B137D466A81C27A487B316BE8/13568)

### 常见负载均衡算法

- 随机
  - 随缘发（用的少）
- 轮询
  - 排队轮询（默认）
- 权重
  - 配置高的机器权重大，概率大
- Hash
  - 通过客户端请求地址，进行 Hash 取模映射调度
- 最小活跃数
  - 发给压力最小的机器

## Ribbon

### 简介

- Spring Cloud Ribbon 是基于 Netflix Ribbon 实现的一套 **客户端按需负载均衡工具**
- 提供一系列的完善配置，如：
  - 超时
  - 重试
  - ...
- 通过 **Load Balancer** 获取到 服务提供的所有实例数据，基于指定规则调用服务

### 模块

- `ribbon-loadbalancer`
  - 负载均衡模块，可单独使用 或 混合使用
- `Ribbon`
  - 内置各种实现的负载均衡算法
- `ribbon-eureka`
  - 基于 Eureka 封装的模块，能快速、方便集成 Eureka
- `ribbon-transport`
  - 基于 Netty 实现多协议的支持，比如 Http、Tcp、Udp ...
- `ribbon-httpclient`
  - 基于 Apache HttpClient 封装的 Rest 客户端，集成负载均衡模块
- `ribbon-example`
  - 使用代码示例
- `ribbon-core`
  - 核心 & 通用 代码

### 使用

```java
/**
 * 演示 Ribbon 如果做负载操作
 * Http调用底层是 HttpURLConnection
 */
public class RibbonDemo {

 		public static void main(String[] args) {
        // 服务列表
        List<Server> serverList = Lists.newArrayList(
                new Server("localhost", 8020),
                new Server("localhost", 8021));

        // 构建负载实例
        ILoadBalancer loadBalancer = LoadBalancerBuilder.newBuilder()
                .buildFixedServerListLoadBalancer(serverList);

        // 调用 5 次来测试效果（服务列表里轮询）
        for (int i = 0; i < 5; i++) {
            String result = LoadBalancerCommand.<String>builder()
                    .withLoadBalancer(loadBalancer).build()
                    .submit(server -> {
                        String addr = "http://" + server.getHost() + ":" +
                                server.getPort() + "/order/findOrderByUserId/1";
                        System.out.println(" 调用地址：" + addr);
                        URL url;
                        try {
                            url = new URL(addr);
                            HttpURLConnection conn = (HttpURLConnection) url.openConnection();
                            conn.setRequestMethod("GET");
                            conn.connect();
                            InputStream in = conn.getInputStream();
                            byte[] data = new byte[in.available()];
                            in.read(data);
                            return Observable.just(new String(data));
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                        return null;
                    }).toBlocking().first();

            System.out.println(" 调用结果：" + result);
        }
        
    }  
  
}
```

### 整合

- 参考上一篇 Fox - Nacos

### 原理

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/13304D5144DD4AD19433C5D084BF365D/13570)

### 模拟Ribbon实现

```java
public class RebbionDemo {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping(value = "/findOrderByUserId/{id}")
    public R  findOrderByUserId(@PathVariable("id") Integer id) {

        //模拟ribbon实现
        String url = getUri("mall-order")+"/order/findOrderByUserId/"+id;

        R result = restTemplate.getForObject(url,R.class);
        return result;
    }

    @Autowired
    private DiscoveryClient discoveryClient;

    public String getUri(String serviceName) {
        // 从注册中心客户端, 通过服务名 获取 服务列表
        List<ServiceInstance> serviceInstances = discoveryClient.getInstances(serviceName);
        
        // 判空
        if (serviceInstances == null || serviceInstances.isEmpty()) {
            return null;
        }
        
        // 服务列表数
        int serviceSize = serviceInstances.size();
        
        // 轮询，获取本次调用下标
        int indexServer = incrementAndGetModulo(serviceSize);
        
        // 返回该下标服务的 URI 属性，其实就是 ip:port
        return serviceInstances.get(indexServer).getUri().toString();
    }
    
    // 下个轮询到的服务下标，初始为0（即第一个服务）
    private AtomicInteger nextIndex = new AtomicInteger(0);
    
    // modulo 是服务列表的 size，其实就是边界值
    private int incrementAndGetModulo(int modulo) {
        // 下面的代码就是 CAS 自旋 更新并获取 最新的服务下标
        for (;;) {
            int current = nextIndex.get();
            int next = (current + 1) % modulo;
            if (nextIndex.compareAndSet(current, next) && current < modulo){
                return current;
            }
        }
    }
    
}
```

### 相关核心接口

- 参考 `RibbonClientConfiguration`

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1621473066605.png)

#### IClientConfig

- Ribbon的客户端配置，默认采用 `DefaultClientConfigImpl` 实现

#### IRule

- Ribbon的负载均衡策略，默认采用 `ZoneAvoidanceRule` 实现
  - 该策略能在 `多区域环境下` 选出 `最佳区域` 的实例进行访问

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/E1376C784BB9439BA29A8CF3105D8E9D/13573)

- `RandomRule`
  - 随机选择一个 Server
- `RetryRule`
  - 对选定的 负载均衡策略 加上 `重试机制`
  - 在一个配置时间段内，当选择 Server 不成功，则一直尝试使用 subRule 的方式选择一个可用服务
- `RoundRobinRule`
  - 轮询选择 Server，轮询下标index，选择 index 对应的 Server
- `AvailabilityFilteringRule`
  - 过滤掉一直连接失败，被标记为 circuit tripped 的Server，并过滤掉高并发的 Server
  - 或使用一个 `AvailabilityPredicate` 来包含过滤 Server 的逻辑
  - 其实就是检查 status 里记录的各个 Server 运行状态
- `WeightedResponseTimeRule`
  - 根据 `响应时间` 加权，响应时间越长、权重越小、被选择的可能性越低
- `ZoneAvoidanceRule（默认）`
  - 复合判断 `Server 所在区域的性能` 和 `Server的可用性` 来选择 Server
  - 在没有区域的环境下，类似于轮询（RandomRule）
- `NacosRule`
  - 同集群优先调用

##### NacosRule核心源码

```java
@Override
	public Server choose(Object key) {
		try {
			String clusterName = this.nacosDiscoveryProperties.getClusterName();
			String group = this.nacosDiscoveryProperties.getGroup();
			DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer) getLoadBalancer();
			String name = loadBalancer.getName();

			NamingService namingService = nacosServiceManager
					.getNamingService(nacosDiscoveryProperties.getNacosProperties());
			List<Instance> instances = namingService.selectInstances(name, group, true);
			if (CollectionUtils.isEmpty(instances)) {
				LOGGER.warn("no instance in service {}", name);
				return null;
			}

			List<Instance> instancesToChoose = instances;
			if (StringUtils.isNotBlank(clusterName)) {
        // 核心逻辑就是这里对统一集群名的实例做了过滤
				List<Instance> sameClusterInstances = instances.stream()
						.filter(instance -> Objects.equals(clusterName,
								instance.getClusterName()))
						.collect(Collectors.toList());
				if (!CollectionUtils.isEmpty(sameClusterInstances)) {
					instancesToChoose = sameClusterInstances;
				}
				else {
					LOGGER.warn(
							"A cross-cluster call occurs，name = {}, clusterName = {}, instance = {}",
							name, clusterName, instances);
				}
			}

			Instance instance = ExtendBalancer.getHostByRandomWeight2(instancesToChoose);

			return new NacosServer(instance);
		}
		catch (Exception e) {
			LOGGER.warn("NacosRule error", e);
			return null;
		}
	}
```

#### IPing

- Ribbon的实例检查策略，默认采用 `DummyPing` 实现
  - 默认策略实际上并不会检查实例是否可用，而是始终返回 true，默认认为所有服务实例皆可用

#### ServerList

- 服务实例清单的维护机制，默认采用 `ConfigurationBasedServerList` 实现

#### ServerListFilter

- 服务实例清单过滤机制，默认采用 `ZonePreferenceServerListFilter` 
  - 该策略能够优先过滤出 `与请求方处于同区域的服务实例`

#### ILoadBalancer

- 负载均衡器，默认采用 `ZoneAwareLoadBalancer` 实现，具备区域感知能力

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/A3D5ED56173D4D2E984014F3E04C2B77/13574)

### 修改默认负载均衡策略

#### 全局配置

```java
@Configuration
public class RibbonConfig {

    @Bean
    public IRule() {
        // 指定使用Nacos提供的负载均衡策略（优先调用同一集群的实例，基于随机权重）
        return new NacosRule();
    }
  
}
```

#### 局部配置

```yaml
# 被调用的微服务名
mall-order:
  ribbon:
    # 指定使用Nacos提供的负载均衡策略（优先调用同一集群的实例，基于随机&权重）
    NFLoadBalancerRuleClassName: com.alibaba.cloud.nacos.ribbon.NacosRule
```

### 自定义负载均衡策略

- 通过实现 `IRule` 接口可以自定义负载策略，主要的选择服务逻辑在 `choose` 方法中

```java
@Slf4j
public class NacosRandomWithWeightRule extends AbstractLoadBalancerRule {

    @Autowired
    private NacosDiscoveryProperties nacosDiscoveryProperties;

    @Override
    public Server choose(Object key) {
        DynamicServerListLoadBalancer loadBalancer = (DynamicServerListLoadBalancer) getLoadBalancer();
        String serviceName = loadBalancer.getName();
        NamingService namingService = nacosDiscoveryProperties.namingServiceInstance();
        try {
            //nacos基于权重的算法
            Instance instance = namingService.selectOneHealthyInstance(serviceName);
            return new NacosServer(instance);
        } catch (NacosException e) {
            log.error("获取服务实例异常：{}", e.getMessage());
            e.printStackTrace();
        }
        return null;
    }


    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {

    }

}
```

### Ribbon的懒加载

- Ribbon 默认懒加载，只有在第一次发起调用的时候才会创建客户端

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/AE0411C4B19F46BA87E6FB5D2F46F62D/13569)

#### 开启饥饿配置

```yaml
# 这个配置对应的源码类是 RibbonEagerLoadProperties
ribbon:
  eager-load:
    # 开启ribbon饥饿加载
    enabled: true
    # 配置mall-user使用ribbon饥饿加载，多个使用逗号分隔，不配即全局饥饿加载
    clients: mall-order
```

![](https://note.youdao.com/yws/public/resource/983c803c0f366af153e5c336aa4ac834/xmlnote/296FC7D9C31F49BCAE00FBC0E19C8CB7/13576)

## LoadBalancer

### 简介

[LoadBalancer的简单介绍](https://note.youdao.com/ynoteshare1/index.html?id=36adba6814ddc01d62363c2a54595a00&type=note)

- Spring Cloud 官方提供的客户端负载均衡器，用于取代 Netflix 的 Ribbon
- 这里记录意义不大，有兴趣可直接参考上面链接

