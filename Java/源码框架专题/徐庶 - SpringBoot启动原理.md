[TOC]

# 徐庶 - SpringBoot启动原理

## 问题

- 为什么 SpringBoot 的 Jar 可以直接运行？
- SpringBoot 是如何启动 Spring 容器的？
- SpringBoot 是如何内置 Tomcat 的？
- 外置 Tomcat 是如何启动 SpringBoot 应用的？
- SPI机制是什么？

## 资料

[知乎 - SpringBoot启动原理](https://zhuanlan.zhihu.com/p/301063931)

[简书 - JarLauncher](https://www.jianshu.com/p/ca5bc7866fb8)

## java -jar 的作用

[Oracle官网原文描述](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/java.html)

```ABAP
If the -jar option is specified, its argument is the name of the JAR file containing class and resource files for the application. The startup class must be indicated by the Main-Class manifest header in its source code.
```

- java -jar 命令后面的参数是 jar 文件名（如app.jar）
  - 该 jar 文件中包含 **class** 和 **资源文件**
  - 在 manifest 文件中有 **Main-Class** 的定义
    - **Main-Class** 的源码中指定了整个应用的启动类

- 即：java -jar 会找到 jar 中的 MANIFEST.MF 文件，从中找到真正的启动类

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

### 问题

- 如果 pom 中没有该插件配置，在 java -jar 时则会抛出异常

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1627876266120.png)

### 核心方法

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

### fat jar的含义

- fat：胖、重量级

- 即：jar in jar ，jar包中还包含着 jar
- 是 java -jar 最终可直接执行的 jar

## jar包目录结构

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

- JarFileArchive 内部的一些依赖 Jar 对应的 URL
  - SpringBoot 使用 org.springframework.boot.loader.jar.Handler 处理器来处理这些 URL

```apl
jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-starter-web-1.3.5.RELEASE.jar!/
```

```apl
jar:file:/Users/Format/Develop/gitrepository/springboot-analysis/springboot-executable-jar/target/executable-jar-1.0-SNAPSHOT.jar!/lib/spring-boot-loader-1.3.5.RELEASE.jar!/org/springframework/boot/loader/JarLauncher.class
```

- 如果 Jar 中包含 Jar 或者 Jar 里面的 class 文件，就使用 **!/** 分开
  - 这种方式只能由上面的 Handler 处理
  - 这是 SpringBoot 内部扩展出来的一种 URL 协议

### JarLauncher

- 从 MANIFEST.MF 中看出，Main函数是 JarLauncher

#### 类结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1627872414831.png)

- 类注释定义：
  - 加载内部 /BOOT-INF/lib 下的 Jar 及 /BOOT-INF/classes 下的应用 class

```java
public abstract class ExecutableArchiveLauncher extends Launcher {

    private final Archive archive;

    public ExecutableArchiveLauncher() {
        try {
            this.archive = createArchive();
        }
        catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
    }

    protected ExecutableArchiveLauncher(Archive archive) {
        this.archive = archive;
    }

    protected final Archive getArchive() {
        return this.archive;
    }

    @Override
    protected String getMainClass() throws Exception {
        Manifest manifest = this.archive.getManifest();
        String mainClass = null;
        if (manifest != null) {
            mainClass = manifest.getMainAttributes().getValue("Start-Class");
        }
        if (mainClass == null) {
            throw new IllegalStateException(
                    "No 'Start-Class' manifest entry specified in " + this);
        }
        return mainClass;
    }

    @Override
    protected List<Archive> getClassPathArchives() throws Exception {
        List<Archive> archives = new ArrayList<>(
                this.archive.getNestedArchives(this::isNestedArchive));
        postProcessClassPathArchives(archives);
        return archives;
    }

    protected abstract boolean isNestedArchive(Archive.Entry entry);

    protected void postProcessClassPathArchives(List<Archive> archives) throws Exception {
    }

}
```

```java
public abstract class Launcher {

    protected void launch(String[] args) throws Exception {
        JarFile.registerUrlProtocolHandler();
        ClassLoader classLoader = createClassLoader(getClassPathArchives());
        launch(args, getMainClass(), classLoader);
    }

    protected ClassLoader createClassLoader(List<Archive> archives) throws Exception {
        List<URL> urls = new ArrayList<>(archives.size());
        for (Archive archive : archives) {
            urls.add(archive.getUrl());
        }
        return createClassLoader(urls.toArray(new URL[0]));
    }

    protected ClassLoader createClassLoader(URL[] urls) throws Exception {
        return new LaunchedURLClassLoader(urls, getClass().getClassLoader());
    }

    protected void launch(String[] args, String mainClass, ClassLoader classLoader)
            throws Exception {
        Thread.currentThread().setContextClassLoader(classLoader);
        createMainMethodRunner(mainClass, args, classLoader).run();
    }

    protected MainMethodRunner createMainMethodRunner(String mainClass, String[] args,
            ClassLoader classLoader) {
        return new MainMethodRunner(mainClass, args);
    }
  
    protected abstract String getMainClass() throws Exception;

    protected abstract List<Archive> getClassPathArchives() throws Exception;

    protected final Archive createArchive() throws Exception {
        ProtectionDomain protectionDomain = getClass().getProtectionDomain();
        CodeSource codeSource = protectionDomain.getCodeSource();
        URI location = (codeSource != null) ? codeSource.getLocation().toURI() : null;
        String path = (location != null) ? location.getSchemeSpecificPart() : null;
        if (path == null) {
            throw new IllegalStateException("Unable to determine code source archive");
        }
        File root = new File(path);
        if (!root.exists()) {
            throw new IllegalStateException(
                    "Unable to determine code source archive from " + root);
        }
        return (root.isDirectory() ? new ExplodedArchive(root)
                : new JarFileArchive(root));
    }

}
```

#### Main工作流程

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1627873559817.png)

1. 新建 JarLauncher 对象并调用父类 Launcher 的 launch 方法启动程序
2. 父类 ExecutableArchiveLauncher 构造中，找到自己所在的 Jar，并创建 Archive
   1. 该无参构造作用就是，创建当前 FatJar 的 JarFileArchive 对象
3. launch() 的作用：
   1. 以 FatJar 为 file 作为入参，构造 JarFileArchive 对象
   2. 获取其中所有资源目标，取得其URL，将这些URL作为参数，构造一个 URLClassLoader
   3. 用该 URLClassLoader 加载 MANIFEST.MF  文件中 Start-Class 指向的业务类，并执行其 Main 方法，进而启动整个 SpringBoot 应用

#### launch()

1. 通过上面 archive.getNestedArchives() 找到 /BOOT-INF/lib 下所有 Jar 以及 /BOOT-INF/classes 下所对应的 archive
2. 通过这些 archives 的 url 生成 LaunchedURLClassLoader，并将其设置为 线程上下文类加载器，启动应用
3. 执行应用程序主类的Main方法
   1. 所有应用程序类文件，均通过 /BOOT-INF/classes 加载
   2. 所有依赖的三方Jar 均通过 /BOOT-INF/lib 加载

### URLStreamHanlder

#### 简介

- Java中，常用 URL 描述资源，
  - URL 打开链接的方法为 java.net.URL # openConnection()
- 因为 URL 可以用于表达各种各样的资源，所以打开资源的具体动作有各种各样的实现，
  - 由 java.net.URLStreamHanlder 该类的子类来完成
  - 根据不同的协议，会由不同的 Handler 实现
  - JDK内置了很多 Handler 用于实现各种协议，如 Jar、File、Http ...
  - URL内部有一个静态 HashTable 属性，用于保存 协议文件 和 对应Handler实例 的映射

#### 三种获取方式

1. 实现 URLStreamHandlerFactory 接口，通过 URL.setURLStreamHandlerFactory 设置
   - 该属性是一个静态属性，且只能被设置一次。
2. 直接提供 URLStreamHandler 的子类，作为 URL 的构造方法的入参之一
   - 但在 JVM 中有固定的规范要求：
     - 子类类名必须是 Handler、最后一级包名必须是协议的名称，如 xx.http.Handler
     - JVM启动时，需要设置 java.protocol.handler.pkgs 系统属性，如果有多个实现类，中间用 | 隔开。
       - 因为 JVM 在尝试寻找 Handler 时，会从该属性中获取包名前缀，最终使用 **包名前缀.协议名.Handler**
       - 使用 Class.forName() 尝试初始化该类，如果初始化成功，则使用该类的实现作为协议实现
3. SpringBoot 为满足 JVM 规范，首先从支持 **jar in jar** 中内容读取做了定制，即支持多个 **!/** 分隔符的 URL 路径
   - **org.springframework.boot.loader.jar.Handler** 继承 **java.net.URLStreamHandler**
     - 该 Handler 支持多个 **!/** 分隔符，并正确打开 URLConnection
     - 打开方式由 **org.springframework.book.loader.jar.JarUrlConnection** 实现
       - 这里还定义了一套读取 ZipFile 的工具类和方法，就不扩展了

## SpringBoot的Jar应用启动流程总结

1. SpringBoot应用打包之后，生成一个fat jar
   - 包含 应用依赖的Jar包、SpringBoot Loader相关类
2. fat jar的启动Main函数是 JarLauncher
   - JarLauncher负责创建一个 LaunchedURLClassLoader 来加载 /lib 下面的 jar
   - 同时开启一个新线程，用来启动应用的Main函数

### LaunchedURLClassLoader

- LaunchedURLClassLoader 实现 ClassLoader
- 具备查找资源和读取资源的能力

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630911742055.png)

#### 读取依赖Jar流程

- SpringBoot构造LaunchedURLClassLoader时，传递了一个 URL[] 数组，数组里就是 lib 目录下面的 Jar 的 URL

1. LaunchedURLClassLoader.loadClass()
2. URL.getContent()
3. URL.openConnection()
4. Handler.openConnection(URL)
5. 最终调用 JarUrlConnection.getInputStream() 

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1630912227647.png)

