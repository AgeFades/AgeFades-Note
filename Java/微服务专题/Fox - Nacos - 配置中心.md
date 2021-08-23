[TOC]

# Fox - Nacos - 配置中心

## 资料

[官方文档](https://github.com/alibaba/spring-cloud-alibaba/wiki/Nacos-config)

[源码脑图](processon.com/view/link/603f3d2fe401fd641adb51f1)

## 简介

- Nacos 提供动态配置功能，可用于配置文件管理、其他KV形式元数据的管理

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/71CAA103D4764215A1A4347F61C72F9A/12925)

## 快速开始

### Nacos控制台新建配置文件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1621564682938.png)

### 引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

### 添加配置

- 新建 bootstrap.yml

```yaml
spring:
	application:
		name: nacos-config
		
	profiles:
		active: dev
		
	cloud:
		nacos:
			config:
				server-addr: 127.0.0.1:8848
				file-extension: yml
				shared-configs:
					- ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
```

### 项目使用配置

```java
@SpringBootApplication
public class NacosConfigApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(NacosConfigApplication.class, args);
        String userName = applicationContext.getEnvironment().getProperty("user.name");
        String userAge = applicationContext.getEnvironment().getProperty("user.age");
        System.out.println("common name :" + userName + "; age: " + userAge);
    }
```

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/7D478250BFE64BFA98730C3F82A7E82A/12926)

### 配置动态更新模拟

```java
@SpringBootApplication
public class NacosConfigApplication {

    public static void main(String[] args) throws InterruptedException {
        ConfigurableApplicationContext applicationContext = SpringApplication.run(NacosConfigApplication.class, args);

         while(true) {
        // 当动态配置刷新时，会更新到 Enviroment中，因此这里每隔一秒中从Enviroment中获取配置
         String userName = applicationContext.getEnvironment().getProperty("common.name");
        String userAge = applicationContext.getEnvironment().getProperty("common.age");
        System.err.println("common name :" + userName + "; age: " + userAge);
            TimeUnit.SECONDS.sleep(1);
        }

    }

}
```

## Config相关配置

- Nacos 数据模型 Key(项目加载的配置类) 由 三元组 唯一确定：
  - `NameSpace`：默认空串，公共命名空间（public）
  - `Group`：分组默认是 DEFAULT_GROUP
  - `DataId`：配置文件名

![](https://note.youdao.com/yws/public/resource/9bf648ba1e0f8dcc4a247c676ea2e72b/xmlnote/63A9D63FE516457781E71F5FACE8A396/14992)

### Profile粒度的配置

- 常用场景是：应用在不同环境下的配置的隔离
- 如：mall-order-dev.yaml、mall-order-prod.yaml

### NameSpace粒度的配置

- 常用场景是：项目在不同环境的配置的隔离
- 不指定则为默认的 Public 命名空间
- 如：mall-dev、mall-prod、erp-dev、erp-prod

### Group粒度的配置

- 常用场景是：项目中不同应用配置的隔离
- 不指定则为默认的 DEFUALT_GROUP  组
- 如: mall-dev 下的 mall-order 组、mall-pay 组

### DataId粒度的配置

- 常用场景是：
  - 多个应用共享同个配置
  - 单个应用使用多个配置

#### 举例

```yaml
spring:
	application:
		name: nacos-config-1
		
	profiles:
		active: dev
		
	cloud:
		nacos:
			config:
				server-addr: 127.0.0.1:8848
				file-extension: yml
				shared-configs:	# 单个应用使用多个配置、下面是数组配置
					- ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
					- common.yaml	# 多个应用共享该配置
```

```yaml
spring:
	application:
		name: nacos-config-2
		
	profiles:
		active: dev
		
	cloud:
		nacos:
			config:
				server-addr: 127.0.0.1:8848
				file-extension: yml
				shared-configs:
					- ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
					- common.yaml
```

### 配置的优先级

- Nacos Config 目前提供三种从 Nacos控制台 拉取配置文件的能力：
  - A：spring.cloud.nacos.config.shared-configs
    - 支持多个共享 DataId 的配置
  - B：spring.cloud.nacos.config.ext-config[n].data-id
    - 支持多个扩展 DataId 的配置
  - C：通过内部规则（应用名、应用名+Profile）自动生成 相关的 DataId 配置
- 优先级为：A < B < C

### @RefreshScope

```java
@RestController
@RefreshScope	// 开启动态感知配置
public class TestController {

  	// @Value 可以获取配置中心的值，但单独使用无法动态刷新
    @Value("${user.age}")
    private String age;

    @GetMapping("/common")
    public String hello() {
        return age;
    }
}
```

### @NacosPropertySource

```java
// 在配置类上加该注解，实现Nacos动态刷新配置
@NacosPropertySource(dataId = "Nacos上的配置文件名", autoRefreshed = true)
```

```java
// 在属性上加该注解
@NacosValue(value = "${user.name:张三}", autoRefreshed = true)
private String userName;
```

