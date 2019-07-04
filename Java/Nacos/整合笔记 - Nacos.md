# 整合笔记 - Nacos

## 简介

​	Nacos : 发现、配置、管理微服务

### 	关键特性

​		服务发现和服务健康监测

​		动态配置服务，带管理界面

​		动态 DNS 服务

​		服务及其元数据管理

## ​下载

​	默认下载 jar 包，本人采用 docker 一键安装部署

```shell
#nacos
docker run -p 8848:8848 \
--name nacos-server \
-e MODE=standalone \	# 单节点启动
--privileged=true \
-v /data/docker/var/lib/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-d docker.io/nacos/nacos-server:1.0.0
```

​	默认路径 : http://localhost:8848/nacos

​	默认账户密码均为 nacos

### 	生产级

​		注意 : 现在全部采用Nacos1.0.0为兼容SpringClou以及Boot2.1

​		创建文件夹挂载目录

```shell
mkdir -p /docker/nacos/conf
mkdir -p /docker/nacos/data
```

​		需 5.7 以上版本 MySQL

```shell
docker run -p 3306:3306 \
--name nacos-mysql \
-e MYSQL_ROOT_PASSWORD=agefades \ # 密码请自定义
--privileged=true \
-v /docker/nacos/data:/var/lib/mysql \
-v /docker/nacos/conf/my.cnf:/etc/mysql/conf.d/mysql.cnf \
-d docker.io/mysql:5.7
```

​		修改字符编码（MySQL 默认拉丁文）

```shell
vim /docker/nacos/conf/my.cnf

# 加入下面内容
[mysqld]
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```

​		进入容器，创建数据库

```shell
docker exec -it nacos-mysql bash

mysql -u root -p
# 输入自定义密码
# 创建数据库
create database nacos;
```

​		创建 nacos 配置文件

```shell
# 首先创建配置文件
vim /docker/nacos/conf/application.properties

# spring
server.servlet.contextPath=${SERVER_SERVLET_CONTEXTPATH:/nacos}
server.contextPath=/nacos
server.port=${NACOS_SERVER_PORT:8848}
spring.datasource.platform=${SPRING_DATASOURCE_PLATFORM:""}
nacos.cmdb.dumpTaskInterval=3600
nacos.cmdb.loadDataAtStart=false


spring.datasource.platform=mysql
db.num=1
# ip : port 看是否需要修改
db.url.0=jdbc:mysql://localhost:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user=root
# 密码需要修改
db.password=agefades


server.tomcat.accesslog.enabled=true
server.tomcat.accesslog.pattern=%h %l %u %t "%r" %s %b %D
# default current work dir
server.tomcat.basedir=
## spring security config
### turn off security

nacos.security.ignore.urls=/,/**/*.css,/**/*.js,/**/*.html,/**/*.map,/**/*.svg,/**/*.png,/**/*.ico,/console-fe/public/**,/v1/auth/login,/v1/console/health/**,/v1/cs/**,/v1/ns/**,/v1/cmdb/**,/actuator/**,/v1/console/server/**
# metrics for elastic search
management.metrics.export.elastic.enabled=fasle
management.metrics.export.influx.enabled=false

nacos.naming.distro.taskDispatchThreadCount=10
nacos.naming.distro.taskDispatchPeriod=200
nacos.naming.distro.batchSyncKeyCount=1000
nacos.naming.distro.initDataRatio=0.9
nacos.naming.distro.syncRetryDelay=5000
nacos.naming.data.warmup=true
```

​		启动容器

```shell
docker run -p 8848:8848 \
--name nacos-server \
-e MODE=standalone \
--privileged=true \
-v /docker/nacos/conf/application.properties:/home/nacos/conf/application.properties \
-d docker.io/nacos/nacos-server:1.0.0
```

![UTOOLS1561962757897.png](https://i.loli.net/2019/07/01/5d19a908a0fdb10427.png)

## 服务注册和发现

​	目前主流的服务注册和发现组件有 : 

​		Consul

​		Eureka（闭源）

​		Etcd ..

### 	构建服务提供者

```xml
<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
			<version>0.9.0.RELEASE</version>
</dependency>
```

```yaml
server:
  port: 8762
spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848	# nacos 服务地址
```

```java
@SpringBootApplication
@EnableDiscoveryClient 	// 启用服务注册发现
public class NacosProviderApplication {

	public static void main(String[] args) {
		SpringApplication.run(NacosProviderApplication.class, args);
	}

}
```

### ​	构建服务消费者

​		同上，略

![UTOOLS1561963052041.png](https://i.loli.net/2019/07/01/5d19aa2d30dc116979.png)

## 服务调用

​	nacos 实现了 Spring Cloud 服务注册和发现的相关接口，所以与其他服务注册与发现组件无缝切换。

​	RestTemplate

​	Feign	

### 	提供服务	

```java
@RestController
@Slf4j
public class ProviderController {

@GetMapping("/hi")
public String hi(@RequestParam(value = "name",defaultValue = "forezp",required = false)String name){

        return "hi "+name;
    }
}

```

### 	消费服务

#### 		RestTemplate

​			使用 Ribbon 作为负载均衡组件

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

```java
	@LoadBalanced	// 启用 Ribbon
	@Bean
	public RestTemplate restTemplate(){
		return new RestTemplate();
	}
```

```java
@RestController
public class ConsumerController {

    @Autowired
    RestTemplate restTemplate;

 	@GetMapping("/hi-resttemplate")
    public String hiResttemplate(){
        return restTemplate.getForObject("http://nacos-provider/hi?name=resttemplate",String.class);
    }
```

#### 		FeignClient

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@EnableFeignClients		// 开启 Feign
public class NacosConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(NacosConsumerApplication.class, args);
	}
```

```java
@FeignClient("nacos-provider")
public interface ProviderClient {

    @GetMapping("/hi")
    String hi(@RequestParam(value = "name", defaultValue = "forezp", required = false) String name);
}
```

```java
@RestController
public class ConsumerController {


    @Autowired
    ProviderClient providerClient;

    @GetMapping("/hi-feign")
    public String hiFeign(){
       return providerClient.hi("feign");
    }
}

```

## 配置中心

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-alibaba-nacos-config</artifactId>
	<version>0.9.0.RELEASE</version>
</dependency>
```

​	bootstrap.yml (不是 application.yml!)

```yaml
spring:
  application:
    name: nacos-provider
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml	# 配置的扩展名
        prefix: nacos-provider	# 配置文件前缀 默认为 spiring.application.name
  profiles:
    active: dev
    # 完整格式 ${prefix}-${spring.profile.active}.${file-extension}
```

![UTOOLS1561963845758.png](https://i.loli.net/2019/07/01/5d19ad48d12d271507.png)

```java
@RestController	
@RefreshScope	// 配置热加载
public class ConfigController {

    @Value("${username:lily}")
    private String username;

    @RequestMapping("/username")
    public String get() {
        return username;
    }
}
```