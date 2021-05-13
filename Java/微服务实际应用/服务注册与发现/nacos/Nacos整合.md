## 简介

- SpringBoot 环境整合 Nacos，实现 **服务注册与发现、动态配置刷新**
- 使用Maven、SpringBoot、SpringCloud、SpingCloudAlibaba 之间版本关系是个大坑

## 依赖

```xml
<properties>
	 	<spring-boot.version>2.3.2.RELEASE</spring-boot.version>
  	<spring-cloud.version>Hoxton.SR8</spring-cloud.version>
  	<spring-cloud-alibaba.version>2.2.5.RELEASE</spring-cloud-alibaba.version>
</properties>

<!-- 其余SpringBoot、Web、AOP之类的依赖自行准备 -->

<dependencyManagement>

	 <!-- SpringCloud 版本依赖控制 -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
                <!-- 排除默认日志包，使用Log4j2 -->
                <exclusions>
                    <exclusion>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-starter-logging</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>

            <!-- SpringCloud Alibaba 版本依赖控制 -->
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
                <!-- 排除默认日志包，使用Log4j2 -->
                <exclusions>
                    <exclusion>
                        <groupId>org.springframework.boot</groupId>
                        <artifactId>spring-boot-starter-logging</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
  
</dependencyManagement>

<dependencies>

	<!-- Nacos 动态配置 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <!-- 排除默认日志包，使用Log4j2 -->
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!-- Nacos 服务注册与发现 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <!-- 排除默认日志包，使用Log4j2 -->
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-logging</artifactId>
                </exclusion>
            </exclusions>
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

