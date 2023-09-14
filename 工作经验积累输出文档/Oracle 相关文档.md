[TOC]

# Oracle 相关文档

## Mac 安装 Oracle

1. 获取 oracle 镜像（3.5G，找其他同事用Mac的隔空投送）

2. docker命令加载镜像

   1. ```shell
      docker load -i oracle.tar
      ```

      ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606294723325.png)

3. docker-compose 启动容器

   1. docker-compose 命令

      1. ```shell
         # 到 oracle-docker-compose 目录下，指定该文件启动
         docker-compose -f oracle-docker-compose.yml up -d
         ```

   2. docker-compose 文件

      1. ```yml
         version: "3.3"
         services:
           oracle:
             image: store/oracle/database-enterprise:12.2.0.1
             container_name: oracle
             hostname: oracle
             restart: always
             environment:
               DB_SID: ORCLCDB
               DB_PDB: ORCLPDB1
               DB_MEMORY: 2GB
               DB_DOMAIN: domain
             volumes:
             	# 挂载文件目录先在自己机器上建好，pwd复制目录修改此处
               - /Users/apple/Documents/Docker/oracle/log:/home/oracle/setup/log
               - /Users/apple/Documents/Docker/oracle/data:/home/oracle/data
             ports:
               - 9080:8080
               - 1521:1521
         ```

   3. docker logs 查看容器日志

      1. ```shell
         # 追踪查看容器日志，出现如下截图表示容器启动成功
         docker logs -f oracle
         ```

         ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606296026432.png)

## 客户端连接

- 本人使用Navicat客户端工具连接Oracle

### 默认超管账号密码

- `账号`：sys
- `密码`：Oradoc_db1

### 连接示例

- `注意`：Service Name、SID、SYSDBA

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606296386449.png)

## 创建账号

```sql
-- 切换容器到pdb
alter session set container=ORCLPDB1;

-- 创建表空间
create tablespace vosp datafile '/home/oracle/data/vosp.dbf' size 5m autoextend on next 5m maxsize 100m;

-- 创建用户
create user vosp identified by vosp default tablespace vosp;

-- 授权
grant create session, connect, dba to vosp;

-- 解锁, 到这一步就可以连接登录了
alter user vosp account unlock;
```

### 连接示例

- `注意`：Service Name、SID、SYSDBA

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606296566055.png)

## Java 整合 Oracle

### 驱动下载

1. 本地Maven settings.xml添加私服配置

   1. ```xml
      <server>
        <id>botpy-nexus</id>
        <!-- 自己的账号密码 -->
        <username>botpy_nexus_username</username>
        <password>botpy_nexus_password</password>
      </server>
      ```

2. IDEA 刷新 Maven、下载驱动jar包

   1. ```xml
      <!-- 使用 oracle -->
      <dependency>
        <groupId>com.oracle.jdbc</groupId>
        <artifactId>ojdbc14</artifactId>
        <version>12.2.0.1</version>
      </dependency>
      ```

3. 修改数据库连接配置

   1. ```yml
      spring:
        datasource:
          # Oracle配置
          url: jdbc:oracle:thin:@127.0.0.1:1521/ORCLPDB1.domain
          driverClassName: oracle.jdbc.driver.OracleDriver
          username: vosp
          password: vosp
      ```

4. 到这里应该就启动成功了

## 事务测试

### 未开启事务

```sql
-- 事务测试
INSERT INTO TEST("id", "name") VALUES (1, '张三');
INSERT INTO TEST("id", "name") VALUES (12345678901, '李四');
INSERT INTO TEST("id", "name") VALUES (3, '王五');
```

#### 执行结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606791567899.png)

- 未开启事务、不会回滚数据

### 开启事务

```sql
-- 事务测试
BEGIN
INSERT INTO TEST("id", "name") VALUES (1, '张三');
INSERT INTO TEST("id", "name") VALUES (12345678901, '李四');
INSERT INTO TEST("id", "name") VALUES (3, '王五');
END;
```

#### 执行结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1606791688968.png)

- 事务内出现异常、执行失败、数据不会提交

## 常用SQL

```sql
-- 查看用户表
select * from dba_users;

-- 查看所有用户
SELECT * FROM ALL_USERS;

-- 修改用户密码
alter user AgeFades identified by 123456;

-- 查看版本
select * from v$version;

-- 查看当前数据库状态
select status from v$instance;

-- 查看当前数据库名称
select name from v$database;

-- 查看服务名
select value from v$parameter where name='service_names';

-- 查看当前数据库中的表空间
SELECT * FROM V$TABLESPACE;

-- 删除空的表空间，但是不包含物理文件
drop tablespace tablespace_name;

-- 删除非空表空间，但是不包含物理文件
drop tablespace tablespace_name including contents;

-- 删除空表空间，包含物理文件
drop tablespace tablespace_name including datafiles;

-- 删除非空表空间，包含物理文件
drop tablespace tablespace_name including contents and datafiles;

-- 创建表空间, 变量如下: 
-- 表空间名{vosp}、
-- 文件路径{/home/oracle/data/vosp.dbf}
-- 大小{SIZE、NEXT、MAXSIZE}
CREATE TABLESPACE vosp
    DATAFILE '/home/oracle/data/vosp.dbf' 
    SIZE 5M 
    AUTOEXTEND ON 
    NEXT 5M 
    MAXSIZE 100M;
    
-- 删除表空间, 变量: 表空间名{vosp}
DROP TABLESPACE vosp;

-- 删除所有表结构
SELECT
	'drop table  "VOSP"."' || table_name || '";' 
FROM
	USER_TABLES 
ORDER BY
	TABLE_NAME;
	
-- 新增字段
-- 新增单个字段
alter table 表名 add (字段名 字段类型 默认值 是否为空);
-- 新增多个字段
alter table 表名 add (字段名 字段类型 默认值 是否为空, 字段名...);

-- 添加表注释
comment on table 表名 is '表注释';

-- 添加字段注释
comment on column 表名.字段名 is '字段注释';

-- 修改字段
alter table 表名 modify (字段名 字段类型 默认值 是否为空);

-- 删除字段
alter table 表名 drop (字段名);

-- 字段重命名
alter table 表名 rename column 旧字段名 to 新字段名;

-- 重命名表
alter table 表名 rename to 新表名

-- 批量插入，字段名要加 ""，值看类型加 ""
insert all
into 表名("字段名1", "字段名2" ...) values ("值1"，"值2" ...)
into 表名("字段名1", "字段名2" ...) values ("值1"，"值2" ...)
into 表名("字段名1", "字段名2" ...) values ("值1"，"值2" ...)
SELECT 1 FROM DUAL;

-- 查询表结构
SELECT
  utc.column_name AS 字段名,
  utc.data_type 数据类型,
  utc.data_length 最大长度,
CASE
    utc.nullable 
    WHEN 'N' THEN
    '否' ELSE '是' 
  END 可空,
  utc.data_default 默认值,
  CASE
  UTC.COLUMN_NAME 
  WHEN ( SELECT col.column_name FROM user_constraints con, user_cons_columns col WHERE con.constraint_name = col.constraint_name AND con.constraint_type = 'P' AND col.table_name = '表名' ) THEN
  '是' ELSE '否' 
  END AS 主键,
  ucc.comments 注释
FROM
  user_tab_columns utc,
  user_col_comments ucc 
WHERE
  utc.table_name = ucc.table_name 
  AND utc.column_name = ucc.column_name 
  AND utc.table_name = '表名' 
ORDER BY
  column_id;
```

## 常见问题

### ORA-00900

#### 错误Msg

```shell
ORA-00900: invalid SQL statement
```

#### 错误原因

- SQL语法错误

#### 解决方案

- 使用正确的SQL语法（google or baidu）

### ORA-00902

#### 错误Msg

```shell
ORA-00902: invalid datatype
```

#### 错误原因

```sql
-- 原SQL、非法数据类型
ALTER TABLE TEST ADD (sex (varchar2(5) null COMMENT '性别');
```

#### 解决方案

```sql
-- 修正语法错误，删除 varchar2 前面的错误括号
ALTER TABLE TEST ADD (sex varchar2(5) null COMMENT '性别');
```

### ORA-00907

#### 错误Msg

```shell
ORA-00907: missing right parenthesis
```

#### 错误原因

```SQL
-- 原错误、错误提示翻译为 缺失右括号
-- 实际上，是向已有表添加字段时，不能直接添加注释
ALTER TABLE TEST ADD (sex varchar2(5) null COMMENT '性别');
```

#### 解决方案

```sql
-- 添加字段
ALTER TABLE TEST ADD (sex varchar2(5) null);

-- 添加注释
COMMENT ON COLUMN TEST.SEX IS '性别';
```

### ORA-01451

#### 错误Msg

```shell
ORA-01451: column to be modified to NULL cannot be modified to NULL
```

#### 错误原因

- oracle 中不允许将 NULL 字段修改为 NULL 字段

```sql
-- 原SQL、将 NULL 字段重复修改为 NULL 字段
ALTER TABLE TEST MODIFY (sex varchar2(10) null);
```

#### 解决方案

```sql
-- 去除重复属性修改即可
ALTER TABLE TEST MODIFY (sex varchar2(10));
```

### ORA-00905

#### 错误Msg

```shell
ORA-00905: missing keyword
```

#### 错误原因

- 缺少关键字

```sql
-- 错误SQL、缺少 into 关键字
INSERT ALL
INSERT TEST(id, name) VALUES (1, '张三')
INSERT TEST(id, name) VALUES (2, '李四')
INSERT TEST(id, name) VALUES (3, '王五')
SELECT 1 FROM DUAL;
```

#### 解决方案

```sql
-- 将 insert 改为 into
INSERT ALL
INTO TEST(id, name) VALUES (1, '张三')
INTO TEST(id, name) VALUES (2, '李四')
INTO TEST(id, name) VALUES (3, '王五')
SELECT 1 FROM DUAL;
```

### ORA-00904

#### 错误Msg

```shell
ORA-00904: "NAME": invalid identifier
```

#### 错误原因

- oracle 中无效的标志符号

```sql
-- 原SQL、字段名没有加 ""
INSERT ALL
INTO TEST(id, name) VALUES (1, '张三')
INTO TEST(id, name) VALUES (2, '李四')
INTO TEST(id, name) VALUES (3, '王五')
SELECT 1 FROM DUAL;
```

#### 解决方案

```sql
-- 字段名加上 ""
INSERT ALL
INTO TEST("id", "name") VALUES (1, '张三')
INTO TEST("id", "name") VALUES (2, '李四')
INTO TEST("id", "name") VALUES (3, '王五')
SELECT 1 FROM DUAL;
```

### ORA-01438

#### 错误Msg

```shell
ORA-01438: value larger than specified precision allowed for this column
```

#### 错误原因

- 插入值要比字段设计的最大长度还大

```sql
-- 原SQL、id 设计为10位Number类型、此处插入 11位
INSERT INTO TEST("id", "name") VALUES (12345678901, '李四');
```

#### 解决方案

```sql
-- 字段值符合字段设计约束
INSERT INTO TEST("id", "name") VALUES (1, '李四');
```

### ORA-01740

#### 错误Msg

```shell
ORA-01740: missing double quote in identifier
```

#### 错误原因

- 缺失的双引号

```sql
-- 原SQL、id 字段缺少左引号
INSERT INTO TEST(id", "name") VALUES (1, '张三');
```

#### 解决方案

```sql
-- 补齐 id 字段引号
INSERT INTO TEST("id", "name") VALUES (1, '张三');
```

## ORA-01788

#### 错误Msg

```shell
ORA-01788: 此查询块中要求 CONNECT BY 子句
```

#### 错误原因

- Java + MyBatisPlus + Oracle 中，需要注意表对应实体字段是否为关键字

- ```java
  /**
   * 级别。brand 品牌，series 车系
   */
  private String level;
  ```

#### 解决方案

```java
/**
 * 级别。brand 品牌，series 车系
 */
@TableField(value = "\"LEVEL\"")
private String level;
```

### ORA-00933

#### 错误Msg

```shell
ORA-00933: SQL 命令未正确结束
```

#### 错误原因

```xml
<!-- mybatis.xml 中，oracle sql 结尾都不能携带 ; -->
<select id="countAllByCustomerId" resultType="java.lang.Integer">
  	SELECT COUNT(*) FROM VEHICLE WHERE CUSTOMER_ID = #{customerId};
</select>
```

#### 解决方案

```xml
<!-- 删除 sql 结尾的 ; -->
<select id="countAllByCustomerId" resultType="java.lang.Integer">
  	SELECT COUNT(*) FROM VEHICLE WHERE CUSTOMER_ID = #{customerId}
</select>
```

### 无法转换为内部表示

#### 错误Msg

```shell
; uncategorized SQLException; SQL state [99999]; error code [17059]; 无法转换为内部表示; nested exception is java.sql.SQLException: 无法转换为内部表示
```

#### 错误原因

1. 检查数据库类型与Java程序接收类型是否一致

   1. [参考链接]: https://blog.csdn.net/jiangyu1013/article/details/53816632

2. Java 中 Mybatis.xml 里，接收对象没有唯一标位（id）

   1. ```xml
       <resultMap id="oilMap" type="com.botpy.vosp.client.oil.repository.YearRecord">
              <id column="year" property="year"/>
              <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.MonthRecord">
                  <id property="month" column="month"/>
                  <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.Record">
                      <!-- 错误XML，这里没有加 id,导致一致报上面的错  -->
         								<!-- <id column="id" property="id" javaType="java.lang.Long"/> -->
                      <result column="name" property="name" jdbcType="VARCHAR"/>
                      <result column="oilValue" property="oilValue"/>
                      <result column="recordTime" property="recordTime"/>
                      <result column="status" property="status"/>
                  </collection>
              </collection>
          </resultMap>
      ```

#### 解决方案

```xml
 <resultMap id="oilMap" type="com.botpy.vosp.client.oil.repository.YearRecord">
        <id column="year" property="year"/>
        <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.MonthRecord">
            <id property="month" column="month"/>
            <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.Record">
                <!-- 正确XML,加上ID标识位 -->
								<id column="id" property="id" javaType="java.lang.Long"/>
                <result column="name" property="name" jdbcType="VARCHAR"/>
                <result column="oilValue" property="oilValue"/>
                <result column="recordTime" property="recordTime"/>
                <result column="status" property="status"/>
            </collection>
        </collection>
    </resultMap>
```

### 数字溢出

#### 错误Msg

```shell
Error attempting to get column 'id' from result set.  Cause: java.sql.SQLException: 数字溢出
```

#### 错误原因

- 数据库中，id 为雪花算法生成，19位，类型为 Number

- Java 中，不指定接收类型，默认与 Number 对应的是 Interger

- [参考链接]: https://www.thinbug.com/q/46642726

- ```xml
  <resultMap id="oilMap" type="com.botpy.vosp.client.oil.repository.YearRecord">
          <id column="year" property="year"/>
          <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.MonthRecord">
              <id property="month" column="month"/>
              <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.Record">
                  <!-- 错误XML,没有指定 id 对应 Long 类型 -->
  								<id column="id" property="id"/>
                  <result column="name" property="name" jdbcType="VARCHAR"/>
                  <result column="oilValue" property="oilValue"/>
                  <result column="recordTime" property="recordTime"/>
                  <result column="status" property="status"/>
              </collection>
          </collection>
      </resultMap>
  ```

#### 解决方案

```xml
<resultMap id="oilMap" type="com.botpy.vosp.client.oil.repository.YearRecord">
        <id column="year" property="year"/>
        <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.MonthRecord">
            <id property="month" column="month"/>
            <collection property="monthRecords" ofType="com.botpy.vosp.client.oil.repository.Record">
                <!-- 正确XML,加上Long类型对应 -->
								<id column="id" property="id" javaType="java.lang.Long"/>
                <result column="name" property="name" jdbcType="VARCHAR"/>
                <result column="oilValue" property="oilValue"/>
                <result column="recordTime" property="recordTime"/>
                <result column="status" property="status"/>
            </collection>
        </collection>
    </resultMap>
```

## 参考资料

[官网](https://www.oracle.com/index.html)

