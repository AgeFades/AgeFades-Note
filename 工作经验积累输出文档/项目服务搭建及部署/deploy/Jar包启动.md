## Maven编译打包

```shell
# 切换目录
cd ~/jdy/JDY-Backend/jdy-parent

# 编译安装
mvn clean install -DskipTest
```

### 异常

 [问题及解决方案](https://blog.csdn.net/sinat_38570489/article/details/89504048)

## Jar包启动

```shell
java -jar jdy-activity/jdy-activity-service/target/jdy-activity-service-1.0-SNAPSHOT.jar
```

### 异常

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1623226498503.png)

#### 解决方案

- 项目pom中添加如下配置

```xml
<build>
        <!--使用项目的artifactId作为docker打包的名称-->
        <finalName>${project.artifactId}</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven-compiler-plugin.version}</version>
                <configuration>
                    <source>${maven.compile.source}</source>
                    <target>${maven.compile.target}</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>${maven-source-plugin.version}</version>
            </plugin>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration> <!-- 指定该Main Class为全局的唯一入口 -->
                    <mainClass>com.botpy.vosp.admin.AdminApiApplication</mainClass>
                    <layout>ZIP</layout>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal><!--可以把依赖的包都打包到生成的Jar包中 -->
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

### 异常

- 找不到common包中的内容

#### 解决方案

- 在被作为 common 依赖的 jar 包 pom 中加如下配置

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-maven-plugin</artifactId>
      <executions>
        <execution>
          <phase>none</phase>
        </execution>
      </executions>
      <configuration>
        <classifier>execute</classifier>
      </configuration>
    </plugin>
  </plugins>
</build>
```

