## 简介

- SpringBoot 环境整合 Nacos，实现 **服务注册与发现、动态配置刷新**
- 使用Maven

## 依赖

```xml
<properties>
	 	<spring-boot.version>2.4.4</spring-boot.version>
    <spring-cloud.version>2020.0.0</spring-cloud.version>
    <spring-cloud-alibaba.version>2021.1</spring-cloud-alibaba.version>
    <spring-cloud-starter-bootstrap.version>3.0.2</spring-cloud-starter-bootstrap.version>
</properties>

<!-- 其余SpringBoot、Web、AOP之类的依赖自行准备 -->

<dependencyManagement>

	<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
  </dependency>

  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
  </dependency>

  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
    <version>${spring-cloud-starter-bootstrap.version}</version>
  </dependency>
  
</dependencyManagement>

<dependencies>

	<!-- Nacos 动态配置 -->
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  </dependency>

  <!-- Nacos 服务注册与发现 -->
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  </dependency>

  <!-- Nacos动态配置 bootstrap 依赖 -->
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
  </dependency>
  
</dependencies>
```

## 配置

### 命名空间隔离

- namespace 命名空间可以起到隔离项目、环境等作用

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1620808548471.png)

### 服务注册与发现

```yaml
# 其他配置自行处理
spring:
	cloud:
    nacos:
      # 服务注册
      discovery:
        server-addr: localhost:8848
        namespace: local
```

