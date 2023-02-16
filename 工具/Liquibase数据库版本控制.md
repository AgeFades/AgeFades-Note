[TOC]

# Liquibase数据库版本控制

## 官网地址

[官方文档](https://www.liquibase.org/)

## 使用背景

有的项目在开发过程中，对SQL的操作记录采用人工方式

1. 开发人员对新需求进行开发，在本地进行库表设计
2. 提测或上线后转给Leader之类的角色，由其在测试或生产环境手动执行

**缺点**

1. 整个项目开发周期，SQL的版本变更无法记录，相关人员无法了解其变更历史
2. 可能存在漏sql、丢sql之类的风险
3. 如果开发人员想做本地开发测试，以减少对公共环境影响，前提为需要项目所有运行所需要的初始表结构和初始表数据（比如：权限表的初始数据）。
   1. 笨办法就是从开发环境导入所有表结构及数据（数据量大的情况下，一堆垃圾数据可能是不想要的，导入过程中可能失败，导入时间巨长...）
   2. 使用数据库版本控制工具，可以通过启动应用，一次执行所有表的建立、数据的初始化，除了点击启动外没有其他操作

## 使用规范

**本人项目实际使用中定义规范，具体使用者可按公司实际情况来定**

1. id作为执行记录表主键,为防止冲突,需严格遵守命名规范:
   1. id = {yyyyMMdd}-{分配的使用数字+当日递增}，举例如下：
   2. 20230105-100(该时间的被分配到100数据段用户的第一个sql变更文件记录)
   3. 20230105-101(该时间的被分配到100数据段用户的第二个sql变更文件记录)
2. author必须如实填写操作用户的真实姓名
3. comments必须填写执行该sql文件的注释以表明其意义
4. 已经被数据库执行过的changeSet严禁修改,每一次执行记录都会用MD5对文件加密存值,用于下次应用启动判断该文件是否被执行过
   1. 所有对已执行过的sql想要回滚或修改,需另外写changeSet进行变更
5. 实际sql文件存储目录分为 data(对数据的操作)/structure(对库表结构的操作)
6. sql文件命名,为防止绝对路径冲突,在最后加上{该目录下同名文件的数量递增}，举例如下：
   1. db/structure/1.0/alter_user.sql
   2. db/structure/1.0/alter_user_1.sql
7. 对表或字段的操作,禁止带上生成的编码及排序规则,统一使用db默认的编码及排序规则
   1. 错误示例：（不同开发人员用不同平台生成出来的SQL，可能导致各个表的编码和排序规则不一致，造成项目开发中各种奇怪的bug）
   2. ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1672890901135.png)

**代码实例示范**

```yaml
# 1.0版本初始SQL执行记录文件
databaseChangeLog:
  - changeSet:
      id: 20230105-100
      author: AgeFades
      comments: 初始化xx项目库表结构及数据
      changes:
        # sql多的可用文件指定
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/structure/1.0/db-structure-init.sql
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/data/1.0/db-data-init.sql
        # sql少的可直接执行sql
        - sql: alter table test add column agefades varchar(255) comment "测试changeSet直接执行sql";
```

## 实际使用案例

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1672890181811.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1672890499446.png)

- z_database_change_log：记录执行记录，及sql文件结合的Md5加密值，用于防止重复执行

```sql
# 截图中该表记录的值
INSERT INTO `z_database_change_log` VALUES ('20230105-100', 'AgeFades', '/db/changelog/changelog-1.0.yaml', '2023-01-05 11:38:28', 1, 'EXECUTED', '8:3337b0417776e4a403bfed53acac9315', 'sqlFile; sqlFile; sql', '', NULL, '3.10.3', NULL, NULL, '2889908227');
```

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <groupId>org.example</groupId>
    <artifactId>log-db-version-control</artifactId>
    <version>1.0-SNAPSHOT</version>
    <modelVersion>4.0.0</modelVersion>
    <description>数据库版本控制</description>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <!-- Import dependency management from Spring Boot -->
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>2.4.5</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <!-- 实现对数据库连接池的自动化配置 -->
        <!-- 同时，spring-boot-starter-jdbc 支持 Liquibase 的自动化配置 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jdbc</artifactId>
        </dependency>

        <!-- 使用 MySQL -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>

        <!-- Liquibase 依赖 -->
        <dependency>
            <groupId>org.liquibase</groupId>
            <artifactId>liquibase-core</artifactId>
        </dependency>
    </dependencies>

</project>
```

### application.yml

```yml
server:
  port: 8003
spring:
  # 数据源配置
  datasource:
    url: jdbc:mysql://localhost:3306/single?useUnicode=true&characterEncoding=UTF-8&useSSL=false
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver

  liquibase:
    enabled: true
    # Liquibase 配置文件地址
    change-log: classpath:/db/changelog/changelog-master.yaml
    database_change_log_table: z_database_change_log
    database_change_log_lock_table: z_database_change_log_lock
    url: ${spring.datasource.url}
    user: ${spring.datasource.username}
    password: ${spring.datasource.password}
```

### 启动类

```java
package com.agefades.db;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * 数据库版本控制应用启动类
 *
 * @author DuChao
 * @since 2023/1/5 10:48
 */
@SpringBootApplication
public class DbVersionControlApplication {
    public static void main(String[] args) {
        SpringApplication.run(DbVersionControlApplication.class, args);
    }
}
```

### changelog-master.yaml

```yaml
databaseChangeLog:
  - include:
      # 1.0版本SQL
      file: /db/changelog/changelog-1.0.yaml
```

### changelog-1.0.yaml

```yaml
# 1.0版本初始SQL执行记录文件
databaseChangeLog:
  - changeSet:
      id: 20230105-100
      author: AgeFades
      comments: 初始化xx项目库表结构及数据
      changes:
        # sql多的可用文件指定
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/structure/1.0/db-structure-init.sql
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/data/1.0/db-data-init.sql
        # sql少的可直接执行sql
        - sql: alter table test add column agefades varchar(255) comment "测试changeSet直接执行sql";
```

### db-structure-init.sql

```sql
CREATE TABLE `test`
(
    `id`   varchar(30) NOT NULL COMMENT '主键',
    `name` varchar(30) COMMENT '是否默认,1是0否,默认0否',
    PRIMARY KEY (`id`)
) ENGINE=InnoDB COMMENT='测试表';
```

### db-data-init.sql

```sql
insert into `test` values ('1', '张三'),('2','李四');
```

## Liquibase & 多数据源管理

### 整体项目文件结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1675654890177.png)

### application.yml

```yaml
server:
  port: 8003
spring:
  # 数据源配置
  datasource:
    single:
      jdbc-url: jdbc:mysql://localhost:3306/single?useUnicode=true&characterEncoding=UTF-8&useSSL=false
      username: root
      password: root
      driverClassName: com.mysql.cj.jdbc.Driver
      liquibase:
        change-log: classpath:/db/changelog/single-changelog-master.yaml

    auth:
      jdbc-url: jdbc:mysql://localhost:3306/auth?useUnicode=true&characterEncoding=UTF-8&useSSL=false
      username: root
      password: root
      driverClassName: com.mysql.cj.jdbc.Driver
      liquibase:
        change-log: classpath:/db/changelog/auth-changelog-master.yaml
```

### single-changelog-master-yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 20230206-100
      author: AgeFades
      comments: 初始化xx项目库表结构及数据
      changes:
        # sql多的可用文件指定
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/structure/1.0/db-structure-init.sql
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/data/1.0/db-data-init.sql
        # sql少的可直接执行sql
        - sql: alter table test add column single varchar(255) comment "测试changeSet直接执行sql";
```

### auth-changelog-master-yaml

```yaml
databaseChangeLog:
  - changeSet:
      id: 20230206-101
      author: AgeFades
      comments: 初始化xx项目库表结构及数据
      changes:
        # sql多的可用文件指定
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/structure/1.0/db-structure-init.sql
        - sqlFile:
            encoding: utf8
            path: classpath:db/sql/data/1.0/db-data-init.sql
        # sql少的可直接执行sql
        - sql: alter table test add column auth varchar(255) comment "测试changeSet直接执行sql";
```

### SingleDsConfig

```java
package com.agefades.db.config;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.liquibase.LiquibaseProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.sql.DataSource;

@Configuration
public class SingleDsConfig {

    @Bean(name = "singleDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.single")
    public DataSource singleDataSource() {
        return DataSourceBuilder.create().build();
    }

    /**
     * 默认使用此事务管理器  如果不想使用 则使用 @Transactional(transactionManager = "transactionManagerWeb") 来指定其他的事务处理器
     * */
    @Bean(name = "transactionManagerSingle")
    @Qualifier(value = "single")
    @Primary
    public DataSourceTransactionManager transactionManagerSingle() {
        return new DataSourceTransactionManager(singleDataSource());
    }

    /**
     * 实现 liquibase在多数据源创建表结构
     * liquibase配置
     * */
    @Bean(name = "singleLiquibaseProperties")
    @ConfigurationProperties(prefix = "spring.datasource.single.liquibase")
    public LiquibaseProperties singleLiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean(name = "singleLiquibase")
    public SpringLiquibase singleLiquibase() {
        return springLiquibase(singleDataSource(), singleLiquibaseProperties());
    }

    private static SpringLiquibase springLiquibase(DataSource dataSource, LiquibaseProperties properties) {
        SpringLiquibase liquibase = new SpringLiquibase();
        liquibase.setDataSource(dataSource);
        liquibase.setChangeLog(properties.getChangeLog());
        liquibase.setContexts(properties.getContexts());
        liquibase.setDefaultSchema(properties.getDefaultSchema());
        liquibase.setDropFirst(properties.isDropFirst());
        liquibase.setShouldRun(properties.isEnabled());
        liquibase.setLabels(properties.getLabels());
        liquibase.setChangeLogParameters(properties.getParameters());
        liquibase.setRollbackFile(properties.getRollbackFile());
        liquibase.setDatabaseChangeLogTable("tbl_database_change_log");
        liquibase.setDatabaseChangeLogLockTable("tbl_database_change_lock_log");
        return liquibase;
    }

}
```

### AuthDsConfig

```java
package com.agefades.db.config;

import liquibase.integration.spring.SpringLiquibase;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.liquibase.LiquibaseProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.jdbc.datasource.DataSourceTransactionManager;

import javax.sql.DataSource;

@Configuration
public class AuthDsConfig {

    @Bean(name = "authDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.auth")
    public DataSource authDataSource() {
        return DataSourceBuilder.create().build();
    }

    /**
     * 默认使用此事务管理器  如果不想使用 则使用 @Transactional(transactionManager = "transactionManagerWeb") 来指定其他的事务处理器
     * */
    @Bean(name = "transactionManagerAuth")
    @Qualifier(value = "auth")
    @Primary
    public DataSourceTransactionManager transactionManagerAuth() {
        return new DataSourceTransactionManager(authDataSource());
    }

    /**
     * 实现 liquibase在多数据源创建表结构
     * liquibase配置
     * */
    @Bean(name = "authLiquibaseProperties")
    @ConfigurationProperties(prefix = "spring.datasource.auth.liquibase")
    public LiquibaseProperties authLiquibaseProperties() {
        return new LiquibaseProperties();
    }

    @Bean(name = "authLiquibase")
    public SpringLiquibase authLiquibase() {
        return springLiquibase(authDataSource(), authLiquibaseProperties());
    }

    private static SpringLiquibase springLiquibase(DataSource dataSource, LiquibaseProperties properties) {
        SpringLiquibase liquibase = new SpringLiquibase();
        liquibase.setDataSource(dataSource);
        liquibase.setChangeLog(properties.getChangeLog());
        liquibase.setContexts(properties.getContexts());
        liquibase.setDefaultSchema(properties.getDefaultSchema());
        liquibase.setDropFirst(properties.isDropFirst());
        liquibase.setShouldRun(properties.isEnabled());
        liquibase.setLabels(properties.getLabels());
        liquibase.setChangeLogParameters(properties.getParameters());
        liquibase.setRollbackFile(properties.getRollbackFile());
        liquibase.setDatabaseChangeLogTable("tbl_database_change_log");
        liquibase.setDatabaseChangeLogLockTable("tbl_database_change_lock_log");
        return liquibase;
    }

}
```

### 测试成功结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1675655006244.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1675655031837.png)
