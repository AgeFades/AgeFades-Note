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

## 参考资料

[官网](https://www.oracle.com/index.html)

