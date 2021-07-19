[TOC]

# 徐庶 - SpringBoot启动原理

## 资料

[知乎 - SpringBoot启动原理](https://zhuanlan.zhihu.com/p/301063931)

## java -jar 的作用

[Oracle官网原文描述](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)

```ABAP
If the -jar option is specified, its argument is the name of the JAR file containing class and resource files for the application. The startup class must be indicated by the Main-Class manifest header in its source code.
```

- java -jar 命令后面的参数是 jar 文件名（如app.jar）
  - 该 jar 文件中包含 **class** 和 **资源文件**
  - 在 manifest 文件中有 **Main-Class** 的定义
    - **Main-Class** 的源码中指定了整个应用的启动类

- 即：java -jar 会找到 jar 中的 manifest 文件，从中找到真正的启动类

### Start-Class

- **MANIFEST.MF** 文件中，有如下内容

```ABAP
Start-Class: com.tulingxueyuan.Application
```

- Start-Class 的值(课程举例)，即 Java项目中真正的应用启动类

### 问题

- 理论上，执行 java -jar 命令时，**JarLauncher**类会被执行
  - 但实际上是 com.tulingxueyuan.Application 被执行了。
- 注意：
  - Java没有提供任何标准的方式，来加载嵌套的jar文件（jar 中依赖的 jar）

## SpringBoot打包插件及核心方法

- Spring Boot 中，pom.xml 默认使用如下插件进行打包

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

- spring-boot-maven-plugin 项目位于 spring-boot-tools 目录中
- 默认有5个goals：
  - repackage
    - 打包默认使用
  - run
  - start
  - stop
  - build-info

### repackage

- 执行 mvn clean package 之后，会生成两个文件（名字是举例）

```ABAP
spring-learn-0.0.1-SNAPSHOT.jar
spring-learn-0.0.1-SNAPSHOT.jar.original
```

- spring-boot-maven-plugin 中的 repackage 的作用：
  - 将 *.jar 再次打包成可执行的软件包
  - 并将 mvn package 生成的软件包重命名为 *.jar.original

#### 源码

- repackage 在代码层面调用了 RepackageMojo 的 execute()，里面调用了 repackage()

```java
private void repackage() throws MojoExecutionException {
  
  // maven生成的jar，最终的命名将加上 .original 后缀
  Artifact source = getSourceArtifact();
  
  // 最终可执行jar，即 fat jar
  File target = getTargetFile();
  
  // 获取重新打包器，将maven生成的jar重新打包成可执行jar
  Repackager repackager = getRepackager(source.getFile());
  
  // 查找并过滤项目运行时依赖的jar
  Set<Artifact> artifacts = filterDependencies(this.project.getArtifacts(), getFilters(getAdditionalFilters()));
  
  // 将 artifacts 转换成 libraries
  Libraries libraries = new ArtifactsLibraries(artifacts, this.requiresUnpack, getLog());
  
  try {
    // 获得 SpringBoot 启动脚本
    LaunchScript launchScript = getLaunchScript();
    
    // 执行重新打包，生成 fat jar
    repackager.repackage(target, libraries, launchScript);
  } catch (IOException ex) {
    throw new MojoExecutionException(ex.getMessage(), ex);
  }
  
  // 将 maven生成的jar，更新成 .original 文件
  updateArtifact(source, target, repackager.getBackupFile());
  
}
```

### jar包目录结构

- 解压 jar

```apl
spring-boot-learn-0.0.1-SNAPSHOT
├── META-INF
│   └── MANIFEST.MF
├── BOOT-INF
│   ├── classes
│   │   └── 应用程序类
│   └── lib
│       └── 第三方依赖jar
└── org
    └── springframework
        └── boot
            └── loader
                └── springboot启动程序
```

### META-INF内容

- 记录 jar包的基础信息、入口程序等..

```apl
Manifest-Version: 1.0
Implementation-Title: spring-learn
Implementation-Version: 0.0.1-SNAPSHOT
Start-Class: com.tulingxueyuan.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.1.5.RELEASE
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

- Main-Class：jar启动的 Main函数
- Start-Class：应用启动的 Main函数

### Archive概念

- archive：递归文档，这个概念在Linux下比较常见
  - 通常就是 tar 或 zip 格式的压缩包
  - jar 是 zip格式
- SpringBoot 中抽象了 Archive 的概念，一个 Archive 可以是：
  - jar：JarFileArchive。
    - 在jar包环境下寻找资源，SpringBoot打包中的fatJar就是使用这个实现
  - 文件目录：ExplodedArchive
    - 在文件夹目录下寻找资源
  - 可以抽象为统一访问资源的逻辑层
- 源码如下：

```java
public interface Archive extends Iterable<Archive.Entry> {
  
  // 获取该归档的 url
  URL getUrl() throws MalformedURLException;
  
  // 获取 jar!/META-INF/MANIFEST.MF 或[ArchiveDir]/META-INF/MANIFEST.MF
  Manifest getManifest() throws IOException;
  
  // 获取 jar!/BOOT-INF/lib/*.jar 或[ArchiveDir]/BOOT-INF/lib/*.jar
  List<Archive> getNestedArchives(EntryFilter filter) throws IOException;
  
}
```

### JarFile

- 对jar包的封装，每个 JarFileArchive 都会对应一个 JarFile
- JarFile被构造时，会解析其内部结构，加载jar包中的各个文件、文件夹并封装到Entry中
  - Entry 也存储在 JarFileArchive 中
  - 如果 Entry 是个Jar，会解析成 JarFileArchive

#### 举例

- 比如一个 JarFileArchive 对应的 URL 为：

```apl
jar:file:/Users/format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/
```

- 它对应的 JarFile 为：

```apl
/Users/format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar
```

- 这个 JarFile 有很多 Entry，比如：

```apl
META-INF/
META-INF/MANIFEST.MF
spring/
spring/study/
....
spring/study/executablejar/ExecutableJarApplication.class
lib/spring-boot-starter-1.3.5.RELEASE.jar
lib/spring-boot-1.3.5.RELEASE.jar
...
```

