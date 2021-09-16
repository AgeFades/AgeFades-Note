[TOC]

# 楼兰 - ShardingJDBC核心理念与快速实战

[官方文档](https://shardingsphere.apache.org/document/5.0.0-beta/en/overview/)

## 简介

- ShardingSphere包含三个重要产品
  - ShardingJDBC
    - 做客户端分库分表的产品
  - ShardingProxy
    - 做服务端分库分表的产品
  - ShardingSidecar
    - 针对 Service Mesh 定位的一个分库分表插件

### ShardingJDBC和ShardingProxy的差异

#### ShardingJDBC

- 在 Java 的 JDBC 层提供额外服务，使用客户端直连数据库
  - 以 Jar 包形式提供服务，无需额外部署和依赖
  - 可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和 各种ORM框架

![](https://note.youdao.com/yws/public/resource/970bbbe5cadce1d30c7ae12801026a26/CD4CAA4F67A046B08F1208D8F14A406C?ynotemdtimestamp=1631525015965)

#### ShardingProxy

- 定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本
  - 目前仅支持 MySQL、PostgreSQL

![](https://note.youdao.com/yws/public/resource/970bbbe5cadce1d30c7ae12801026a26/BBB0DB635B044CEB8EE133FB9B312658?ynotemdtimestamp=1631525015965)

#### 差异

|            | Sharding-JDBC | Sharding-Proxy   |
| ---------- | ------------- | ---------------- |
| 数据库     | 任意          | MySQL/PostgreSQL |
| 连接消耗数 | 高            | 低               |
| 异构语言   | 仅java        | 任意             |
| 性能       | 损耗低        | 损耗略高         |
| 无中心化   | 是            | 否               |
| 静态入口   | 无            | 有               |

- ShardingJDBC 只是客户端的一个工具包，所有分库分表逻辑均由业务方自己控制
  - 功能相对灵活，支持数据库多
  - 但是对业务代码侵入大
- ShardingProxy是一个独立部署的服务，
  - 对业务无侵入
  - 功能固定，不方便灵活扩展
  - 支持数据库少

## 实战

- ShardingJDBC 核心功能：
  - 数据分片
  - 读写分离

### 核心概念

- 逻辑表：
  - 水平拆分的数据库的 **相同逻辑和数据结构** 表的总称
- 真实表：
  - 在分片数据库中，真实存在的物理表
- 数据节点：
  - 数据分片的最小单元，由数据源名称和数据表组成
- 绑定表：
  - 分片规则一致的主表和子表
- 广播表（公共表）：
  - 所有分片数据源中都存在的表
  - 表结构和数据在每个节点上都完全一致
  - 例如：字典表
- 分片键：
  - 用于分片的数据库字段
  - SQL中若没有分片字段，将会执行全路由（性能差）
- 分片算法：
  - 对数据进行分片的算法逻辑，如 取模、取余...
  - 支持通过 =、between、in 分片
  - 由开发者自行实现
- 分片策略：
  - 分片键 + 分片算法 = 分片策略
  - 一般采用基于 Groovy 表达式的 inline 分片策略
    - 如t_user_$->{u_id%8}标识根据u_id模8，分成8张表，表名称为t_user_0到t_user_7

### 分片策略

#### None

- 不分片

#### Inline

- 最常用的分片方式，按照分片表达式来进行分片
- 配置参数：
  - inline.shardingColumn：分片键
  - inline.algorithmExpression：分片表达式

#### Standard

- 只支持单个分片键的标准分片策略
- 配置参数：
  - standard.shardingColumn：分片键
  - standard.precise-algorithm-class-name：精准分片算法实现类名
  - standard.range-algorithm-class-name：范围分片算法实现类名
- 实现方式：
  - 精准分片：实现 **PreciseShardingAlgorithm** 接口
    - 提供 = 或 in 的逻辑
    - 必须提供
  - 范围分片：实现 **RangeShardingAlgorithm** 接口
    - 提供 between 的逻辑
    - 可选提供

#### Complex

- 支持多分片键的复杂分片策略

- 配置参数：
  - complex.sharding-columns：分片键(多个)
  - complex.algorithm-class-name：分片算法实现类
- 实现方式：
  - 实现 **ComplexKeysShardingAlgorithm** 接口
  - 提供按照 **多个分片键** 进行综合分片的算法

#### Hint

- 不需要分片键的强制分片策略
- 配置参数：
  - hint.algorithm-class-name： 分片算法实现类
- 实现方式：
  - 实现 **HintShardingAlgorithm** 接口
  - 该算法类中，同样需要分片键
    - 通过 HintManager.addDatabaseShardingValue() 指定分库
    - 通过 HintManager.addTableShardingValue() 指定分表
- 注意：
  - 这里分片键是线程隔离的，只在当前线程有效
  - 通常建议使用之后立即关闭，或用 try 资源方式打开
- 优点：
  - Hint 没有完全按照 SQL 解析树来构建分片策略，是绕开了 SQL 解析的
  - 所以对某些复杂SQL，Hint 性能可能会比较好
- 缺点：
  - Hint 强制路由在使用时，有很多限制

```sql
-- 不支持UNION
SELECT * FROM t_order1 UNION SELECT * FROM t_order2
INSERT INTO tbl_name (col1, col2, …) SELECT col1, col2, … FROM tbl_name WHERE col3 = ?

-- 不支持多层子查询
SELECT COUNT(*) FROM (SELECT * FROM t_order o WHERE o.id IN (SELECT id FROM t_order WHERE status = ?))

-- 不支持函数计算。ShardingSphere只能通过SQL字面提取用于分片的值
SELECT * FROM t_order WHERE to_date(create_time, 'yyyy-mm-dd') = '2019-01-01';
```

### 功能实测

- entity、mapper、service、serviceImpl、启动类 都是最简单常规的，自己建就好

#### SQL

```sql
CREATE TABLE course_1
(
    cid     BIGINT(20) PRIMARY KEY,
    cname   VARCHAR(50) NOT NULL,
    user_id BIGINT(20) NOT NULL,
    cstatus varchar(10) NOT NULL
);

CREATE TABLE course_2
(
    cid     BIGINT(20) PRIMARY KEY,
    cname   VARCHAR(50) NOT NULL,
    user_id BIGINT(20) NOT NULL,
    cstatus varchar(10) NOT NULL
);

-- 在三个库中创建
CREATE TABLE `dict`
(
    `dict_id` bigint(0) PRIMARY KEY NOT NULL,
    `ustatus` varchar(100) NOT NULL,
    `uvalue`  varchar(100) NOT NULL
);
-- 在userdb中创建
CREATE TABLE `dict_1`
(
    `dict_id` bigint(0) PRIMARY KEY NOT NULL,
    `ustatus` varchar(100) NOT NULL,
    `uvalue`  varchar(100) NOT NULL
);
CREATE TABLE `dict_2`
(
    `dict_id` bigint(0) PRIMARY KEY NOT NULL,
    `ustatus` varchar(100) NOT NULL,
    `uvalue`  varchar(100) NOT NULL
);

CREATE TABLE `user`
(
    `user_id`  bigint(0) PRIMARY KEY NOT NULL,
    `username` varchar(100) NOT NULL,
    `ustatus`  varchar(50)  NOT NULL,
    `uage`     int(3)
);

CREATE TABLE `user_1`
(
    `user_id`  bigint(0) PRIMARY KEY NOT NULL,
    `username` varchar(100) NOT NULL,
    `ustatus`  varchar(50)  NOT NULL,
    `uage`     int(3)
);
CREATE TABLE `user_2`
(
    `user_id`  bigint(0) PRIMARY KEY NOT NULL,
    `username` varchar(100) NOT NULL,
    `ustatus`  varchar(50)  NOT NULL,
    `uage`     int(3)
);
```

#### pom

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.0.RELEASE</version>
        <relativePath/>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>sharding-jdbc-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <java.version>11</java.version>
        <mybatis-plus.version>3.3.2</mybatis-plus.version>
        <hutool.version>5.5.7</hutool.version>
        <mysql.version>8.0.17</mysql.version>
        <sharding-jdbc.version>4.1.1</sharding-jdbc.version>
    </properties>

    <dependencies>
        <!-- SpringMVC -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring 单测 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Hutool -->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>${hutool.version}</version>
        </dependency>

        <!-- mybatis plus -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>

        <!-- mysql驱动 -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>

        <!-- JDK11所需依赖 -->
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
            <version>2.3.1</version>
        </dependency>

        <!-- ShardingJDBC -->
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
            <version>${sharding-jdbc.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

#### Inline分表单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 分表规则
      tables:
        # 针对 course 表的分片规则
        course:
          # 真实数据节点,采用Groovy表达式
          # 这里表示 ds1 数据源的 course_1 和 course_2 表
          actual-data-nodes: ds1.course_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: cid
            # 雪花算法生成
            type: SNOWFLAKE
          # 分表策略
          table-strategy:
            # 使用 inline 策略
            inline:
              # 分片键
              sharding-column: cid
              # 分片算法表达式
              # 这里表示通过 cid % 2 + 1 算式,决定数据落到 course_1 或 course_2
              algorithm-expression: course_$->{cid%2+1}
```

##### 代码

```java
/**
 * Inline 分表策略测试, 对应 application-1.yml
 *
 * @author DuChao
 * @date 2021/9/14 11:50 上午
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class ShardingJDBCTest1 {

    @Resource
    CourseService courseService;

    /**
     * 测试1. Course 分表插入测试
     * 测试结论: 代码批量插入，实际执行了10次插入SQL
     * 均匀分布到两张表中，分表测试成功
     */
    @Test
    public void addCourse() {
        List<Course> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(Course.builder()
                    .cname("ShardingSphere")
                    .userId(Convert.toLong(i))
                    .cstatus("1")
                    .build());
        }
        courseService.saveBatch(list);
    }

    /**
     * 测试2. Course 分表查询测试（无条件）
     * 测试结论：实际执行了两次SQL，将两表数据归并响应（全路由）
     */
    @Test
    public void courseList() {
        List<Course> list = courseService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试3. Course 分表查询测试（非路由键 = 条件测试）
     * 测试结论：实际执行了两次SQL，将两表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByUserId() {
        Course course = courseService.lambdaQuery().eq(Course::getUserId, 1).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试4. Course 分表查询测试（路由键 = 条件测试）
     * 测试结论：实际执行了1次SQL，先根据 cid 条件路由到真实表，再执行查询
     */
    @Test
    public void queryCourseByCid() {
        Course course = courseService.lambdaQuery().eq(Course::getCid, 644498172680339457L).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试5. Course 分表查询测试（路由键 between 范围条件测试）
     * 测试结论：Inline strategy cannot support this type sharding:RangeRouteValue
     *  Inline 策略不支持 路由键范围查询
     */
    @Test
    public void queryCourseByBetweenCid() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getCid, 644498172680339457L, 644498172718088192L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试6. Course 分表查询测试（非路由键 between 范围条件测试）
     * 测试结论：实际执行了两次SQL，将两表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenUserId() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getUserId, 1, 10)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试7. Course 分表查询测试（路由键 in 条件测试）
     * 测试结论：先根据 cid 条件路由到真实表，再执行查询，都在一个表就只实行一次，在不同表就执行多次
     */
    @Test
    public void queryCourseByInCid() {
        List<Course> list = courseService.lambdaQuery()
                .in(Course::getCid, 644498172340600832L, 644498172718088192L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### Inline分库分表单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1,ds2
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3307/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 分表规则
      tables:
        # 针对 course 表的分片规则
        course:
          # 真实数据节点,采用Groovy表达式
          # 这里表示 ds1 数据源的 course_1 和 course_2 表
          actual-data-nodes: ds$->{1..2}.course_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: cid
            # 雪花算法生成
            type: SNOWFLAKE
          # 分表策略
          table-strategy:
            # 使用 inline 策略
            inline:
              # 分片键
              sharding-column: cid
              # 分片算法表达式
              # 这里表示通过 cid % 2 + 1 算式,决定数据落到 course_1 或 course_2
              algorithm-expression: course_$->{cid%2+1}
          # 分库策略
          database-strategy:
            inline:
              # 分片键
              sharding-column: cid
              # 分片算法表达式
              # 这里表示通过 cid % 2 + 1 算式,决定数据落到 course_1 或 course_2
              algorithm-expression: ds$->{cid%2+1}
```

##### 代码

```java
/**
 * Inline 分库分表测试, 对应 application-inline2.yml
 *
 * @author DuChao
 * @date 2021/9/14 11:50 上午
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class InlineTest2 {

    @Resource
    CourseService courseService;

    /**
     * 测试1. Course 分表插入测试
     * 测试结论: 代码批量插入，实际执行了10次插入SQL
     * 均匀分布到 ds1_course_1、ds2_course_2 中，分库分表测试成功
     */
    @Test
    public void addCourse() {
        List<Course> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(Course.builder()
                    .cname("ShardingSphere")
                    .userId(Convert.toLong(i))
                    .cstatus("1")
                    .build());
        }
        courseService.saveBatch(list);
    }

    /**
     * 测试2. Course 分表查询测试（无条件）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void courseList() {
        List<Course> list = courseService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试3. Course 分表查询测试（非路由键 = 条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByUserId() {
        Course course = courseService.lambdaQuery().eq(Course::getUserId, 1).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试4. Course 分表查询测试（路由键 = 条件测试）
     * 测试结论：实际执行了1次SQL，先根据 cid 条件路由到真实表，再执行查询
     */
    @Test
    public void queryCourseByCid() {
        Course course = courseService.lambdaQuery().eq(Course::getCid, 645202679093526529L).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试5. Course 分表查询测试（路由键 between 范围条件测试）
     * 测试结论：Inline strategy cannot support this type sharding:RangeRouteValue
     *  Inline 策略不支持 路由键范围查询
     */
    @Test
    public void queryCourseByBetweenCid() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getCid, 645202679093526529L, 645202679106109441L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试6. Course 分表查询测试（非路由键 between 范围条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenUserId() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getUserId, 1, 10)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试7. Course 分表查询测试（路由键 in 条件测试）
     * 测试结论：先根据 cid 条件路由到真实表，再执行查询，都在一个表就只实行一次，在不同表就执行多次
     */
    @Test
    public void queryCourseByInCid() {
        List<Course> list = courseService.lambdaQuery()
                .in(Course::getCid, 645202679093526529L, 645202679106109441L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### Standard单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1,ds2
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3307/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 分表规则
      tables:
        # 针对 course 表的分片规则
        course:
          # 真实数据节点,采用Groovy表达式
          # 这里表示 ds1 数据源的 course_1 和 course_2 表
          actual-data-nodes: ds$->{1..2}.course_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: cid
            # 雪花算法生成
            type: SNOWFLAKE
          # 分表策略
          table-strategy:
            # 使用 standard 策略
            standard:
              # 分片键
              sharding-column: cid
              precise-algorithm-class-name: com.agefades.demo.algorithm.StandardTableAlgorithm
              range-algorithm-class-name: com.agefades.demo.algorithm.StandardTableAlgorithm
          # 分库策略
          database-strategy:
            standard:
              # 分片键
              sharding-column: cid
              precise-algorithm-class-name: com.agefades.demo.algorithm.StandardDsAlgorithm
              range-algorithm-class-name: com.agefades.demo.algorithm.StandardDsAlgorithm
```

##### 代码

```java
/**
 * Standard策略分库算法实现类
 *
 * @author DuChao
 * @date 2021/9/16 10:16 上午
 */
public class StandardDsAlgorithm implements PreciseShardingAlgorithm<Long>, RangeShardingAlgorithm<Long> {

    /**
     * @param availableTargetNames 可用的目标库名称集合(此例中即 ds1、ds2)
     * @param preciseShardingValue 路由键的精准值(= or in)
     * @return 路由导向的目标库名称
     */
    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> preciseShardingValue) {
        // 模拟SQL：select * from course where cid = ? or cid in (?,?)
        // 获取逻辑表名称
        String logicTableName = preciseShardingValue.getLogicTableName();
        // 获取分片键字段名
        String cid = preciseShardingValue.getColumnName();
        // 获取分片键值
        Long value = preciseShardingValue.getValue();

        // 根据分片键值,定制分片算法,实现真实库路由（这里简单的取模）
        String dsName = "ds" + (value % 2 + 1);
        if (!availableTargetNames.contains(dsName)) {
            throw new UnsupportedOperationException("路由库" + dsName + "不存在，请检查配置");
        }
        return dsName;
    }

    /**
     * @param availableTargetNames 可用的目标库名称集合(此例中即 ds1、ds2)
     * @param rangeShardingValue   路由键的范围值(between)
     * @return 路由导向的目标库名称集合
     */
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> rangeShardingValue) {
        // 模拟SQL：select * from course where cid between 1 and 100;
        // 获取上边界值 100
        Long upper = rangeShardingValue.getValueRange().upperEndpoint();
        // 获取下边界值 1
        Long lower = rangeShardingValue.getValueRange().lowerEndpoint();
        // 获取逻辑表名称
        String logicTableName = rangeShardingValue.getLogicTableName();

        // 范围查询直接全库全表路由
        return availableTargetNames;
    }
}
```

```java
public class StandardTableAlgorithm implements PreciseShardingAlgorithm<Long>, RangeShardingAlgorithm<Long> {

    /**
     * @param availableTargetNames 可用的目标表名称集合(此例中即 course_1、course_2)
     * @param preciseShardingValue 路由键的精准值(= or in)
     * @return 路由导向的目标表名称
     */
    @Override
    public String doSharding(Collection<String> availableTargetNames, PreciseShardingValue<Long> preciseShardingValue) {
        // 模拟SQL：select * from course where cid = ? or cid in (?,?)
        // 获取逻辑表名称
        String logicTableName = preciseShardingValue.getLogicTableName();
        // 获取分片键字段名
        String cid = preciseShardingValue.getColumnName();
        // 获取分片键值
        Long value = preciseShardingValue.getValue();

        // 根据分片键值,定制分片算法,实现真实表路由（这里简单的取模）
        String tableName = logicTableName + "_" + (value % 2 + 1);
        if (!availableTargetNames.contains(tableName)) {
            throw new UnsupportedOperationException("路由表" + tableName + "不存在，请检查配置");
        }
        return tableName;
    }

    /**
     * @param availableTargetNames 可用的目标表名称集合(此例中即 course_1、course_2)
     * @param rangeShardingValue   路由键的范围值(between)
     * @return 路由导向的目标表名称集合
     */
    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, RangeShardingValue<Long> rangeShardingValue) {
        // 模拟SQL：select * from course where cid between 1 and 100;
        // 获取上边界值 100
        Long upper = rangeShardingValue.getValueRange().upperEndpoint();
        // 获取下边界值 1
        Long lower = rangeShardingValue.getValueRange().lowerEndpoint();
        // 获取逻辑表名称
        String logicTableName = rangeShardingValue.getLogicTableName();

        // 范围查询直接全表路由
        return availableTargetNames;
    }
}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class StandardTest {

    @Resource
    CourseService courseService;

    /**
     * 测试1. Course 分表插入测试
     * 测试结论: 代码批量插入，实际执行了10次插入SQL
     * 均匀分布到 ds1_course_1、ds2_course_2 中，分库分表测试成功
     */
    @Test
    public void addCourse() {
        List<Course> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(Course.builder()
                    .cname("ShardingSphere")
                    .userId(Convert.toLong(i))
                    .cstatus("1")
                    .build());
        }
        courseService.saveBatch(list);
    }

    /**
     * 测试2. Course 分表查询测试（无条件）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void courseList() {
        List<Course> list = courseService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试3. Course 分表查询测试（非路由键 = 条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByUserId() {
        Course course = courseService.lambdaQuery().eq(Course::getUserId, 1).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试4. Course 分表查询测试（路由键 = 条件测试）
     * 测试结论：实际执行了1次SQL，先根据 cid 条件路由到真实表，再执行查询
     */
    @Test
    public void queryCourseByCid() {
        Course course = courseService.lambdaQuery().eq(Course::getCid, 645211700114489345L).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试5. Course 分表查询测试（路由键 between 范围条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenCid() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getCid, 645211700114489345L, 645211700127072257L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试6. Course 分表查询测试（非路由键 between 范围条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenUserId() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getUserId, 1, 10)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试7. Course 分表查询测试（路由键 in 条件测试）
     * 测试结论：先根据 cid 条件路由到真实表，再执行查询，都在一个表就只实行一次，在不同表就执行多次
     */
    @Test
    public void queryCourseByInCid() {
        List<Course> list = courseService.lambdaQuery()
                .in(Course::getCid, 645202679093526529L, 645202679106109441L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### Complex单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1,ds2
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 分表规则
      tables:
        # 针对 course 表的分片规则
        course:
          # 真实数据节点,采用Groovy表达式
          # 这里表示 ds1 数据源的 course_1 和 course_2 表
          actual-data-nodes: ds$->{1..2}.course_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: cid
            # 雪花算法生成
            type: SNOWFLAKE
          # 分表策略
          table-strategy:
            # 使用 standard 策略
            complex:
              # 分片键
              sharding-columns: cid, user_id
              algorithm-class-name: com.agefades.demo.algorithm.ComplexTableAlgorithm
          # 分库策略
          database-strategy:
            complex:
              # 分片键
              sharding-columns: cid, user_id
              algorithm-class-name: com.agefades.demo.algorithm.ComplexDsAlgorithm
```

##### 代码

```java
/**
 * Complex策略分库算法实现类
 *
 * @author DuChao
 * @date 2021/9/16 10:16 上午
 */
public class ComplexDsAlgorithm implements ComplexKeysShardingAlgorithm<Long> {

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, ComplexKeysShardingValue<Long> shardingValue) {
        // 多分片键

        // cid 可能为 范围 或 精确值 或 精确值的集合（所以下面用 Collection 包起来）
        Range<Long> cidRange = shardingValue.getColumnNameAndRangeValuesMap().get("cid");
        Collection<Long> cidCol = shardingValue.getColumnNameAndShardingValuesMap().get("cid");

        // userId 可能为 范围 或 精确值 或 精确值的集合（所以下面用 Collection 包起来）
        Range<Long> userIdRange = shardingValue.getColumnNameAndRangeValuesMap().get("user_id");
        Collection<Long> userIdCol = shardingValue.getColumnNameAndShardingValuesMap().get("user_id");

        // 如果有对分片键的范围查询,应该走全路由
        if (cidRange != null || userIdRange != null) {
            return availableTargetNames;
        }

        // 决定分片路由规则（这里简单模拟，优先以 c_id 做分片规则, 其次以 userId 做分片的规则）
        if (cidCol != null) {
            return cidCol.stream()
                    .map(cid -> "ds" + (cid % 2 + 1))
                    .collect(Collectors.toList());
        } else {
            return userIdCol.stream()
                    .map(userId -> "ds" + (userId % 2 + 1))
                    .collect(Collectors.toList());
        }
    }

}
```

```java
/**
 * Complex策略分表算法实现类
 *
 * @author DuChao
 * @date 2021/9/16 10:16 上午
 */
public class ComplexTableAlgorithm implements ComplexKeysShardingAlgorithm<Long> {

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, ComplexKeysShardingValue<Long> shardingValue) {
        // 多分片键

        // cid 可能为 范围 或 精确值 或 精确值的集合（所以下面用 Collection 包起来）
        Range<Long> cidRange = shardingValue.getColumnNameAndRangeValuesMap().get("cid");
        Collection<Long> cidCol = shardingValue.getColumnNameAndShardingValuesMap().get("cid");

        // userId 可能为 范围 或 精确值 或 精确值的集合（所以下面用 Collection 包起来）
        Range<Long> userIdRange = shardingValue.getColumnNameAndRangeValuesMap().get("user_id");
        Collection<Long> userIdCol = shardingValue.getColumnNameAndShardingValuesMap().get("user_id");

        // 如果有对分片键的范围查询,应该走全路由
        if (cidRange != null || userIdRange != null) {
            return availableTargetNames;
        }

        // 决定分片路由规则（这里简单模拟，优先以 c_id 做分片规则, 其次以 userId 做分片的规则）
        if (cidCol != null) {
            return cidCol.stream()
                    .map(cid -> shardingValue.getLogicTableName() + "_" + (cid % 2 + 1))
                    .collect(Collectors.toList());
        } else {
            return userIdCol.stream()
                    .map(userId -> shardingValue.getLogicTableName() + "_" + (userId % 2 + 1))
                    .collect(Collectors.toList());
        }
    }

}
```

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class ComplexTest {

    @Resource
    CourseService courseService;

    /**
     * 测试1. Course 分表插入测试
     * 测试结论: 代码批量插入，实际执行了10次插入SQL
     * 均匀分布到 ds1_course_1、ds2_course_2 中，分库分表测试成功
     */
    @Test
    public void addCourse() {
        List<Course> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(Course.builder()
                    .cname("ShardingSphere")
                    .userId(Convert.toLong(i))
                    .cstatus("1")
                    .build());
        }
        courseService.saveBatch(list);
    }

    /**
     * 测试2. Course 分表查询测试（无条件）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void courseList() {
        List<Course> list = courseService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试3. Course 分表查询测试（路由键 = 条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByUserId() {
        Course course = courseService.lambdaQuery().eq(Course::getUserId, 1L).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试4. Course 分表查询测试（路由键 = 条件测试）
     * 测试结论：实际执行了1次SQL，先根据 cid 条件路由到真实表，再执行查询
     */
    @Test
    public void queryCourseByCid() {
        Course course = courseService.lambdaQuery().eq(Course::getCid, 645221038996586496L).one();
        System.out.println(JSONUtil.toJsonStr(course));
    }

    /**
     * 测试5. Course 分表查询测试（路由键 between 范围条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenCid() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getCid, 645221038996586496L, 645221039155970048L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试6. Course 分表查询测试（路由键 between 范围条件测试）
     * 测试结论：实际执行了四次SQL，将两库四表数据归并响应（全路由）
     */
    @Test
    public void queryCourseByBetweenUserId() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getUserId, 1L, 10L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试7. Course 分表查询测试（路由键 in 条件测试）
     * 测试结论：先根据 cid 条件路由到真实表，再执行查询，都在一个表就只实行一次，在不同表就执行多次
     */
    @Test
    public void queryCourseByInCid() {
        List<Course> list = courseService.lambdaQuery()
                .in(Course::getCid, 645221038996586496L, 645221039155970048L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试8. Course 分表查询测试（多路由键条件测试）
     * 测试结论：有范围查询即全路由
     */
    @Test
    public void queryCourseByBetweenCidAndEqUserId() {
        List<Course> list = courseService.lambdaQuery()
                .between(Course::getCid, 645221038996586496L, 645221039155970048L)
                .eq(Course::getUserId, 1L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

    /**
     * 测试9. Course 分表查询测试（多路由键条件测试）
     * 测试结论：先根据 cid 条件路由到真实表，再执行查询，都在一个表就只实行一次，在不同表就执行多次
     */
    @Test
    public void queryCourseByInCidAndEqUserId() {
        List<Course> list = courseService.lambdaQuery()
                .in(Course::getCid, 645221038996586496L, 645221039155970048L)
                .eq(Course::getUserId, 1L)
                .list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### Hint单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1,ds2
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 分表规则
      tables:
        # 针对 course 表的分片规则
        course:
          # 真实数据节点,采用Groovy表达式
          # 这里表示 ds1 数据源的 course_1 和 course_2 表
          actual-data-nodes: ds$->{1..2}.course_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: cid
            # 雪花算法生成
            type: SNOWFLAKE
          # 分表策略
          table-strategy:
            # 使用 hint 策略
            hint:
              # 分片键
              algorithm-class-name: com.agefades.demo.algorithm.HintTableAlgorithm
          # 分库策略
          database-strategy:
          	# 使用 standard
            standard:
              # 分片键
              sharding-column: cid
              precise-algorithm-class-name: com.agefades.demo.algorithm.StandardDsAlgorithm
              range-algorithm-class-name: com.agefades.demo.algorithm.StandardDsAlgorithm
```

##### 代码

```java
/**
 * Hint策略分表算法实现类
 *
 * @author DuChao
 * @date 2021/9/16 10:16 上午
 */
public class HintTableAlgorithm implements HintShardingAlgorithm<Long> {

    @Override
    public Collection<String> doSharding(Collection<String> availableTargetNames, HintShardingValue<Long> shardingValue) {
        // 强制指定路由表
        Collection<Long> values = shardingValue.getValues();
        if (values != null) {
            return values.stream()
                    .map(value -> shardingValue.getLogicTableName() + "_" + value)
                    .filter(availableTargetNames::contains)
                    .collect(Collectors.toList());
        }
        return availableTargetNames;
    }

}
```



```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class HintTest {

    @Resource
    CourseService courseService;

    /**
     * 测试1. Course 分表查询测试（强制指定路由表）
     * 测试结论：这里是不同策略组合，Standard 分库策略 + Hint 分表策略
     *  所以这里查 ds1.course_2 和 ds2.course_2
     */
    @Test
    public void queryCourseByHint() {
        // 强制指定查 course_2 表
        HintManager.getInstance().addTableShardingValue("course", 2L);
        List<Course> list = courseService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### 广播表单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds1,ds2
      # 数据源1配置
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      ds2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      broadcast-tables: dict
      tables:
        dict:
          # 主键生成规则
          key-generator:
            # 主键名
            column: dict_id
            # 雪花算法生成
            type: SNOWFLAKE
```

##### 代码

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class BroadCastTest {

    @Resource
    DictService dictService;

    /**
     * 测试1. Dict 广播表插入数据
     */
    @Test
    public void addDict() {
        dictService.save(Dict.builder()
                .ustatus("1")
                .uvalue("正常")
                .build());

        dictService.save(Dict.builder()
                .ustatus("2")
                .uvalue("不正常")
                .build());
    }

    /**
     * 测试2. Dict 广播表查数据
     * 测试结论：会随机找一个数据源查数据（这里测的都是查 ds2）
     */
    @Test
    public void dictList() {
        List<Dict> list = dictService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### 绑定表单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: ds
      # 数据源1配置
      ds:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 配置绑定表
      binding-tables:
        - user,dict
      tables:
        dict:
          actual-data-nodes: ds.dict_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: dict_id
            # 雪花算法生成
            type: SNOWFLAKE
          table-strategy:
            inline:
              sharding-column: ustatus
              algorithm-expression: dict_$->{ustatus.toInteger()%2+1}
        user:
          actual-data-nodes: ds.user_$->{1..2}
          # 主键生成规则
          key-generator:
            # 主键名
            column: user_id
            # 雪花算法生成
            type: SNOWFLAKE
          table-strategy:
            inline:
              sharding-column: ustatus
              algorithm-expression: user_$->{ustatus.toInteger()%2+1}
```

##### 代码

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class BindingTest {

    @Resource
    DictService dictService;

    @Resource
    UserService userService;

    /**
     * 测试1. 插入Dict、User测试数据
     * 测试结果: 实际执行12条sql，正确分表插入
     */
    @Test
    public void addTestData() {
        dictService.save(Dict.builder()
                .ustatus("1")
                .uvalue("正常")
                .build());

        dictService.save(Dict.builder()
                .ustatus("2")
                .uvalue("不正常")
                .build());

        List<User> list = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            list.add(User.builder()
                    .username("user" + i)
                    .uage(i * 10)
                    .ustatus((i % 2 + 1) + "")
                    .build());
        }
        userService.saveBatch(list);
    }

    /**
     * 测试2. 两个分片表连表查
     * 测试结论：
     * 1. 没有配置 user 和 dict 的绑定表时，执行了4次sql，做了笛卡尔乘积
     * 2. 配置了绑定表时，执行了2次sql
     * 注意：这里要是出现 IndexOutOfBing 数组越界
     * 是 User 加了 @Builder 还要加 @NoArgsConstruct 和
     * @AllArgsConsturct
     */
    @Test
    public void userList() {
        // @Select("select u.user_id, u.username, d.uvalue ustatus from user u left join dict d on u.ustatus = d.ustatus")
        List<User> list = userService.queryUserStatus();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```

#### 读写分离单测

##### 配置

```yaml
spring:
  main:
    # 允许 bean 定义覆盖
    allow-bean-definition-overriding: true

  # 分库分表配置
  shardingsphere:
    # 属性项配置
    props:
      sql:
        # 打印SQL
        show: true

    datasource:
      names: m0,s1
      # 数据源1配置
      m0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root
      # 数据源2配置
      s1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/syncdemo?serverTimezone=GMT%2B8
        username: root
        password: root

    sharding:
      # 读写分离、主写从读
      master-slave-rules:
        ds0:
          master-data-source-name: m0
          slave-data-source-names:
            - s1
      # 分表规则
      tables:
        dict:
          actual-data-nodes: ds0.dict
          # 主键生成规则
          key-generator:
            # 主键名
            column: dict_id
            # 雪花算法生成
            type: SNOWFLAKE

```

##### 代码

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class RwSeparateTest {

    @Resource
    DictService dictService;

    /**
     * 测试1. 插入Dict测试数据
     * 测试结果: 插入两条sql, 都是往 m0 主库插
     */
    @Test
    public void addTestData() {
        dictService.save(Dict.builder()
                .ustatus("1")
                .uvalue("正常")
                .build());

        dictService.save(Dict.builder()
                .ustatus("2")
                .uvalue("不正常")
                .build());
    }

    /**
     * 测试2. 查 dict 数据
     * 测试结论: 查1条sql，从 s1 从库查
     */
    @Test
    public void dictList() {
        List<Dict> list = dictService.list();
        System.out.println(JSONUtil.toJsonStr(list));
    }

}
```



## SQL使用限制

[官方文档各版本支持SQL类型](https://shardingsphere.apache.org/document/current/cn/features/sharding/use-norms/sql/)

**支持的SQL**

| SQL                                                          | 必要条件                                |
| :----------------------------------------------------------- | :-------------------------------------- |
| SELECT * FROM tbl_name                                       |                                         |
| SELECT * FROM tbl_name WHERE (col1 = ? or col2 = ?) and col3 = ? |                                         |
| SELECT * FROM tbl_name WHERE col1 = ? ORDER BY col2 DESC LIMIT ? |                                         |
| SELECT COUNT(*), SUM(col1), MIN(col1), MAX(col1), AVG(col1) FROM tbl_name WHERE col1 = ? |                                         |
| SELECT COUNT(col1) FROM tbl_name WHERE col2 = ? GROUP BY col1 ORDER BY col3 DESC LIMIT ?, ? |                                         |
| INSERT INTO tbl_name (col1, col2,…) VALUES (?, ?, ….)        |                                         |
| INSERT INTO tbl_name VALUES (?, ?,….)                        |                                         |
| INSERT INTO tbl_name (col1, col2, …) VALUES (?, ?, ….), (?, ?, ….) |                                         |
| INSERT INTO tbl_name (col1, col2, …) SELECT col1, col2, … FROM tbl_name WHERE col3 = ? | INSERT表和SELECT表必须为相同表或绑定表  |
| REPLACE INTO tbl_name (col1, col2, …) SELECT col1, col2, … FROM tbl_name WHERE col3 = ? | REPLACE表和SELECT表必须为相同表或绑定表 |
| UPDATE tbl_name SET col1 = ? WHERE col2 = ?                  |                                         |
| DELETE FROM tbl_name WHERE col1 = ?                          |                                         |
| CREATE TABLE tbl_name (col1 int, …)                          |                                         |
| ALTER TABLE tbl_name ADD col1 varchar(10)                    |                                         |
| DROP TABLE tbl_name                                          |                                         |
| TRUNCATE TABLE tbl_name                                      |                                         |
| CREATE INDEX idx_name ON tbl_name                            |                                         |
| DROP INDEX idx_name ON tbl_name                              |                                         |
| DROP INDEX idx_name                                          |                                         |
| SELECT DISTINCT * FROM tbl_name WHERE col1 = ?               |                                         |
| SELECT COUNT(DISTINCT col1) FROM tbl_name                    |                                         |
| SELECT subquery_alias.col1 FROM (select tbl_name.col1 from tbl_name where tbl_name.col2=?) subquery_alias |                                         |

**不支持的SQL**

| SQL                                                          | 不支持原因                                                   |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| INSERT INTO tbl_name (col1, col2, …) VALUES(1+2, ?, …)       | VALUES语句不支持运算表达式                                   |
| INSERT INTO tbl_name (col1, col2, …) SELECT * FROM tbl_name WHERE col3 = ? | SELECT子句暂不支持使用*号简写及内置的分布式主键生成器        |
| REPLACE INTO tbl_name (col1, col2, …) SELECT * FROM tbl_name WHERE col3 = ? | SELECT子句暂不支持使用*号简写及内置的分布式主键生成器        |
| SELECT * FROM tbl_name1 UNION SELECT * FROM tbl_name2        | UNION                                                        |
| SELECT * FROM tbl_name1 UNION ALL SELECT * FROM tbl_name2    | UNION ALL                                                    |
| SELECT SUM(DISTINCT col1), SUM(col1) FROM tbl_name           | 详见DISTINCT支持情况详细说明                                 |
| SELECT * FROM tbl_name WHERE to_date(create_time, ‘yyyy-mm-dd’) = ? | 会导致全路由                                                 |
| (SELECT * FROM tbl_name)                                     | 暂不支持加括号的查询                                         |
| SELECT MAX(tbl_name.col1) FROM tbl_name                      | 查询列是函数表达式时,查询列前不能使用表名;若查询表存在别名,则可使用表的别名 |

**DISTINCT支持情况详细说明**

**支持的SQL**

| SQL                                                          |
| :----------------------------------------------------------- |
| SELECT DISTINCT * FROM tbl_name WHERE col1 = ?               |
| SELECT DISTINCT col1 FROM tbl_name                           |
| SELECT DISTINCT col1, col2, col3 FROM tbl_name               |
| SELECT DISTINCT col1 FROM tbl_name ORDER BY col1             |
| SELECT DISTINCT col1 FROM tbl_name ORDER BY col2             |
| SELECT DISTINCT(col1) FROM tbl_name                          |
| SELECT AVG(DISTINCT col1) FROM tbl_name                      |
| SELECT SUM(DISTINCT col1) FROM tbl_name                      |
| SELECT COUNT(DISTINCT col1) FROM tbl_name                    |
| SELECT COUNT(DISTINCT col1) FROM tbl_name GROUP BY col1      |
| SELECT COUNT(DISTINCT col1 + col2) FROM tbl_name             |
| SELECT COUNT(DISTINCT col1), SUM(DISTINCT col1) FROM tbl_name |
| SELECT COUNT(DISTINCT col1), col1 FROM tbl_name GROUP BY col1 |
| SELECT col1, COUNT(DISTINCT col1) FROM tbl_name GROUP BY col1 |

**不支持的SQL**

| SQL                                                          | 不支持原因                                                   |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| SELECT SUM(DISTINCT tbl_name.col1), SUM(tbl_name.col1) FROM tbl_name | 查询列是函数表达式时,查询列前不能使用表名;若查询表存在别名,则可使用表的别名 |

## 简单总结

1. 能不用分库分表，就不用分库分表
2. 要用，就要在系统之处设计好
3. MySQL传统关系型数据库，就存点关系型强的数据
   1. 海量数据可以用 PostGreSQL、VoltDB、HBase、Hive、ES 等等其他产品来做
4. OLTP：
   1. 是传统的关系型数据库的主要应用，主要是基本的、日常的事务处理，例如银行交易
5. OLAP：
   1. 是数据仓库系统的主要应用，支持复杂的分析操作，侧重决策支持，并且提供直观易懂的查询结果

## 设计案例

- 电商商品管理模块大致如下

![](https://note.youdao.com/yws/public/resource/970bbbe5cadce1d30c7ae12801026a26/04AADBC5314E49978B051A40984DA27D?ynotemdtimestamp=1631525015965)

1. 以业务为单位考虑对数据进行垂直分片，店铺、产品、商品三种业务数据垂直拆分成三个不同的库。字典表作为广播表冗余到三个不同的库中
2. 考虑数据增长情况，商品将会是以后增长最快的数据，店铺和产品的数据增速会逐渐降低。所以对商品表进行分片。分片策略采用商品ID取模的方式，尽量保证商品数据平均分片
3. 将关联性较强的商品信息表和商品补充信息表配置为绑定表

![](https://note.youdao.com/yws/public/resource/970bbbe5cadce1d30c7ae12801026a26/C1850C2735194F8599603A630EFD1E48?ynotemdtimestamp=1631525015965)

