### 安装

```shell
sudo apt install maven
```

### 验证

```shell
mvn -v
```

### Maven家路径

```shell
cd /usr/share/maven
```

### 验证Java

```shell
java -version
```

### Maven配置文件

```shell
# 编辑配置文件
sudo vim /usr/share/maven/conf/settings.xml
```

```xml
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
  <!-- localRepository
   | The path to the local repository maven will use to store artifacts.
   |
   | Default: ${user.home}/.m2/repository
  <localRepository>/path/to/local/repo</localRepository>
  -->

  <pluginGroups>

  </pluginGroups>

  <proxies>

  </proxies>

  <servers>

    <!-- 博派通达Nexus -->
    <server>
        <id>botpy-releases</id>
        <username>duchao</username>
        <password>Ycj-uSZL!]v]_jX\</password>
    </server>
    <server>
        <id>botpy-snapshots</id>
        <username>duchao</username>
        <password>Ycj-uSZL!]v]_jX\</password>
    </server>
    <server>
        <id>botpy-nexus</id>
        <username>duchao</username>
        <password>Ycj-uSZL!]v]_jX\</password>
    </server>
  </servers>

  <mirrors>
     <!-- 阿里云 -->
     <mirror>
        <id>alimaven</id>
        <name>aliyun maven</name>
        <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
        <mirrorOf>central</mirrorOf>
    </mirror>

    <!-- 博派通达Nexus -->
     <mirror>
      <id>botpy-nexus</id>
      <mirrorOf>*</mirrorOf>
      <url>https://nxrm.botpy.com/repository/maven-public/</url>
    </mirror>
  </mirrors>

  <profiles>
      <profile>
        <id>botpy-private-repo</id>
        <repositories>
            <repository>
                <id>botpy-releases</id>
                <url>https://nxrm.botpy.com/repository/maven-releases/</url>
            </repository>
            <repository>
                <id>botpy-snapshots</id>
                <url>https://nxrm.botpy.com/repository/maven-snapshots/</url>
            </repository>
        </repositories>
      </profile>
  </profiles>

</settings>

```

