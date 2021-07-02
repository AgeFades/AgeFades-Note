[TOC]



# Fox - Nacos - 配置中心源码分析

[源码流程图](https://www.processon.com/view/link/60d4312de401fd50b991b583)

## 问题

1. 配置刷新，Client端是如何动态感知的？
2. 多个配置优先级是怎样的？
3. 集群节点是如何同步配置的？

## 架构图

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/F30997AA23404B00A492D8D4D06E92C4/15455)

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/3827B56135FA45879C6FCDB80CAFBA77/14998)

## Nacos Client入口

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

#### 应用初始化器

- **ApplicationContextInitializer**
  - Spring 的扩展点之一，在 ConfigurableApplication # refresh() 之前调用
  - 通用用于需要对应用上下文做初始化的 Web 应用中，比如：
    - 根据上下文环境注册属性源 或 激活配置文件等

#### SPI机制加载应用初始化器

[SPI资料](https://segmentfault.com/a/1190000039812642)

- 在 spring-cloud-context 包中实现，通过 SPI 机制加载
- 在 SpringApplication 初始化的过程中，会通过 SpringFactoriesLoader 加载 ApplicationContextInitializer 类型的实现类到 initializers 中

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624416702364.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624416793137.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624416938860.png)

#### 属性配置源初始化器的初始化方法

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624417327461.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624417401875.png)

#### 拉取Nacos中配置文件的优先级

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624417880558.png)

- **locate()** 加载顺序:
  1. 共享资源配置
  2. 扩展资源配置
  3. 应用资源配置
- **locate()** 优先级: 3 > 2 > 1

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624418506820.png)

- **loadApplicationConfiguration()** 加载顺序:
  1. 文件名（服务名）
  2. 文件名.扩展名
  3. 文件名-profile.扩展名
- **loadApplicationConfiguration() **优先级：3 > 2 > 1

#### 调用Nacos ConfigService接口

- 调用栈

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624435785014.png)

#### 优先加载本地配置文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624435945078.png)

#### 本地没有则发送Http请求

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624436140278.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624436197657.png)

##### 集群下选择Server发送接口的实现

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624436790168.png)

#### 保存本地快照

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624436610278.png)

### 注册监听器

#### 监听IOC容器启动完成

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624868116315.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624868245445.png)

#### 监听器遍历配置文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624868334317.png)

#### 依次注册配置刷新监听器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624868663879.png)

#### 配置刷新监听器的处理

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624869262115.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624869437524.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624869512669.png)

- 之前 @RefreshScope 标记的 bean 被 destroy 销毁后
  - 下次应用获取这些 Bean，就会重新从 BeanFactory 中创建 Bean
  - 此时该 Bean 就是最新的配置Bean了

## Nacos 中配置中心的顶级核心接口

- **ConfigService**

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624434833065.png)

### 代码案例

```java
public class ConfigServerDemo {

    public static void main(String[] args) throws NacosException, InterruptedException {
        String serverAddr = "localhost:8848";
        String dataId = "nacos-config-demo.yaml";
        String group = "DEFAULT_GROUP";
        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        //获取配置服务
        ConfigService configService = NacosFactory.createConfigService(properties);
        //获取配置
        String content = configService.getConfig(dataId, group, 5000);
        System.out.println(content);
        //注册监听器
        configService.addListener(dataId, group, new Listener() {
            @Override
            public void receiveConfigInfo(String configInfo) {
                System.out.println("===recieve:" + configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        });

        //发布配置
        //boolean isPublishOk = configService.publishConfig(dataId, group, "content");
        //System.out.println(isPublishOk);
        //发送properties格式
        configService.publishConfig(dataId,group,"common.age=30", ConfigType.PROPERTIES.getType());

        Thread.sleep(3000);
        content = configService.getConfig(dataId, group, 5000);
        System.out.println(content);

//        boolean isRemoveOk = configService.removeConfig(dataId, group);
//        System.out.println(isRemoveOk);
//        Thread.sleep(3000);

//        content = configService.getConfig(dataId, group, 5000);
//        System.out.println(content);
//        Thread.sleep(300000);

    }
}
```

## Nacos Server响应拉取配置请求

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624436390229.png)

### 本地配置文件读取

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624871290006.png)

- DiskUtil.targetFile() 也是从本地磁盘读取配置文件的方法
  - 课程 debug 进入的该方法

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624871380285.png)

- 直接改数据库配置文件的数据，是不能实现客户端配置动态刷新的
  - 因为拉取配置时，Nacos Server都是直接从本地磁盘读的数据
- Console修改配置，需要发布 ConfigDataChangeEvent 的事件
  - 触发本地文件和内存的更新

## Nacos Server启动时，将数据库中的配置数据dump到本地磁盘

### 顶级抽象服务类

- Nacos  Server中的**DumpService**

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624871941160.png)

### 应用启动初始化

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872049255.png)

### 先清空配置历史数据

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872378417.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872439997.png)

### 进行读库写盘

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872524636.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872661509.png)

### 更新文件md5

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872831817.png)

### 发布配置数据变更事件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624872968491.png)

## Nacos Console更新并发布配置文件

### 接收接口

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624873144672.png)

### 核心函数

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624873248587.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624873362941.png)

### 发布事件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624873574294.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1624873748101.png)

