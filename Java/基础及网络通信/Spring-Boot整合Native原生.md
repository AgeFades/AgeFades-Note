# 什么是SpringNative？

​		GraalVM 是一种高性能运行时，可显着提高应用程序性能和效率，非常适合微服务. 对于 Java 程序 GraalVM 负责将 Java 字节码编译成机器码，映像生成过程使用静态分析来查找可从主 Java 方法访问的任何代码，然后执行完全提前 (AOT) 编译。生成的本机二进制文件包含机器代码形式的整个程序，以便立即执行。

​		Spring Native 支持使用[GraalVM ](https://www.graalvm.org/)[native-image](https://www.graalvm.org/reference-manual/native-image/)编译器将 Spring 应用程序编译为本机可执行文件。

​		与 Java 虚拟机相比，本机映像可以为多种类型的工作负载提供更便宜且更可持续的托管。这些包括微服务、功能工作负载，非常适合容器和[Kubernetes](https://kubernetes.io/)。

​		简单的来说就是直接将Java字节码编译成二进制文件，直接执行，可以启动的非常快，然后在云原生的适配以及镜像上非常方便。

​		官方文档地址：[点击进入](https://docs.spring.io/spring-native/docs/current/reference/htmlsingle/)

# 和JVM的区别？

​		使用本机映像提供了关键优势，例如即时启动、即时峰值性能和减少内存消耗。

​		GraalVM 原生项目也有一些缺点和权衡，预计会随着时间的推移而改进。构建本机映像是一个繁重的过程，比常规应用程序要慢。原生映像在预热后的运行时优化较少。最后，它在一些不同的行为方面不如 JVM 成熟。

​		常规 JVM 和本机映像平台之间的主要区别是：

- ​					在构建时从主入口点对您的应用程序进行静态分析。
- ​					未使用的部分在构建时被删除。
- ​					反射、资源和动态代理需要配置。
- ​					类路径在构建时是固定的。
- ​					无类延迟加载：可执行文件中的所有内容都将在启动时加载到内存中。
- ​					一些代码将在构建时运行。
- ​					Java 应用程序的某些方面存在一些不完全支持的限制。

​		该项目的目标是孵化对 Spring Native（Spring JVM 的替代方案）的支持，并提供旨在打包在轻量级容器中的原生部署选项。实际上，目标是在这个新平台上支持几乎未修改的 Spring 应用程序。







```shell
cat > ./docker-compose.yaml << EOF
version: '3.4'
services:
  spring-native:
    container_name: spring-native       # 指定容器的名称
    image: docker.io/library/test-native:0.0.1-SNAPSHOT     # 指定镜像和版本
    restart: always  # 自动重启
    hostname: native
    ports:
      - 8080:8080
    privileged: true
EOF
```









```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.2</version>
        <relativePath/>
    </parent>
    <groupId>test.native</groupId>
    <artifactId>test-native</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>test-native</name>
    <description>test-native</description>
    <properties>
        <java.version>1.8</java.version>
        <!-- springboot生成jar文件的文件名后缀，用来避免Spring Boot repackaging和native-image-maven-plugin插件之间可能存在的冲突 -->
        <classifier/>

        <!-- 构建镜像时的定制参数 -->
        <native.build.args>
            --no-fallback
            --initialize-at-build-time=org.springframework.util.unit.DataSize
            --initialize-at-build-time=org.slf4j.MDC
            --initialize-at-build-time=ch.qos.logback.classic.Level
            --initialize-at-build-time=ch.qos.logback.classic.Logger
            --initialize-at-build-time=ch.qos.logback.core.util.StatusPrinter
            --initialize-at-build-time=ch.qos.logback.core.status.StatusBase
            --initialize-at-build-time=ch.qos.logback.core.status.InfoStatus
            --initialize-at-build-time=ch.qos.logback.core.spi.AppenderAttachableImpl
            --initialize-at-build-time=org.slf4j.LoggerFactory
            --initialize-at-build-time=ch.qos.logback.core.util.Loader
            --initialize-at-build-time=org.slf4j.impl.StaticLoggerBinder
            --initialize-at-build-time=ch.qos.logback.classic.spi.ThrowableProxy
            --initialize-at-build-time=ch.qos.logback.core.CoreConstants
            --report-unsupported-elements-at-runtime
            --allow-incomplete-classpath
            -H:+ReportExceptionStackTraces
        </native.build.args>

        <!-- 指定使用dmikusa/graalvm-tiny这个镜像作为构建工具，来构建镜像 -->
        <builder>dmikusa/graalvm-tiny</builder>


    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!--        <dependency>-->
        <!--            <groupId>org.apache.tomcat.experimental</groupId>-->
        <!--            <artifactId>tomcat-embed-programmatic</artifactId>-->
        <!--            <version>${tomcat.version}</version>-->
        <!--        </dependency>-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.experimental</groupId>
            <artifactId>spring-native</artifactId>
            <version>0.10.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <!-- 插件管理 -->
    <pluginRepositories>
        <pluginRepository>
            <id>spring-release</id>
            <name>Spring release</name>
            <url>https://repo.spring.io/release</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
        <pluginRepository>
            <id>spring-milestone</id>
            <name>Spring milestone</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </pluginRepository>
        <pluginRepository>
            <id>spring-snapshot</id>
            <name>Spring Snapshots</name>
            <url>https://repo.spring.io/snapshot</url>
            <releases>
                <enabled>false</enabled>
            </releases>
        </pluginRepository>
    </pluginRepositories>

    <!--仓库管理-->
    <repositories>
        <repository>
            <id>spring-release</id>
            <name>Spring release</name>
            <url>https://repo.spring.io/release</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-milestone</id>
            <name>Spring milestone</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
        <repository>
            <id>spring-snapshot</id>
            <name>Spring Snapshots</name>
            <url>https://repo.spring.io/snapshot</url>
            <releases>
                <enabled>false</enabled>
            </releases>
        </repository>
    </repositories>


    <!--    &lt;!&ndash;依赖包版本管理&ndash;&gt;-->
    <!--    <dependencyManagement>-->
    <!--        <dependencies>-->
    <!--            <dependency>-->
    <!--                <groupId>org.springframework.experimental</groupId>-->
    <!--                <artifactId>spring-native</artifactId>-->
    <!--                <version>0.10.0-SNAPSHOT</version>-->
    <!--            </dependency>-->
    <!--        </dependencies>-->
    <!--    </dependencyManagement>-->


    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <mainClass>com.test.nati.TestNativeApplication</mainClass>
                    <classifier>${classifier}</classifier>
                    <image>
                        <builder>${builder}</builder>
                        <env>
                            <BP_NATIVE_IMAGE>true</BP_NATIVE_IMAGE>
                            <BP_NATIVE_IMAGE_BUILD_ARGUMENTS>${native.build.args}</BP_NATIVE_IMAGE_BUILD_ARGUMENTS>
                        </env>
                        <!--执行构建任务的镜像，如果在当前环境不存在才会远程下载-->
                        <pullPolicy>IF_NOT_PRESENT</pullPolicy>
                    </image>
                </configuration>
            </plugin>
            <!-- aot插件，ahead-of-time transformations -->
            <plugin>
                <groupId>org.springframework.experimental</groupId>
                <artifactId>spring-aot-maven-plugin</artifactId>
                <version>0.10.0-SNAPSHOT</version>
                <executions>
                    <execution>
                        <id>test-generate</id>
                        <goals>
                            <goal>test-generate</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>generate</id>
                        <goals>
                            <goal>generate</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>

```

