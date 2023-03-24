[TOC]

# 朱晓峰 - MySQL必知必会

## 0. 环境准备 | 安装MySQL

[官方下载地址](https://dev.mysql.com/downloads/mysql/)

[MySQL官方论坛](https://forums.mysql.com/)

- 课程介绍了Windows下的安装过程，本人使用电脑为Mac系统，因此仅记录安装过程中对MySQL的一些介绍
- 课程安装的图形化界面工具是 MySQL 自带的 MySQL Workbench，本人使用 Navicat，因此不做记录

### 第一步：安装MySQL数据库服务器及相关组件

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679641323615.png)

- 介绍这些关键组件的作用如下：
  1. MySQL Server：是MySQL数据库服务器，是MySQL的核心组件
  2. MySQL  Workbench：是一个管理MySQL的图形工具
  3. MySQL Shell：是一个命令行工具。除了支持SQL语句，它还支持 JavaScript 和 Python 脚本，并且支持调用 MySQL  API 接口
  4. MySQL  Router：是一个轻量级的插件，可以在应用和数据库服务器之间，起到路由和负载均衡的作用。
     - 想象一个场景：假设有多个 MySQL 数据库服务器，而前端的应用同时产生了很多数据库访问请求，这时，MySQL  Router 就可以对这些请求进行调度，把访问均衡地分配个每个数据库服务器，而不是集中在一个或几个数据库服务器上。
  5. Connector/ODBC：是 MySQL  数据库的 ODBC 驱动程序。ODBC 是微软的一套数据库连接标准，微软的产品（比如Excel）就可以通过 ODBC 驱动与 MySQL  数据库连接。
- 其它的组件，主要用来支持各种开发环境与 MySQL 的连接，还有 MySQL 帮助文档和示例，见名知意。

### 第二步：配置服务器

- 组件安装完之后，会提示配置服务器的类型（Config Type）、连接（Connectivity）以及高级选项（Advanced Configuration）等

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679641728131.png)

- 主要需要配置2个部分，分别是 **服务器类别** 和 **服务器连接**

#### 服务器类别配置

- 我们有3个选项，它们的区别在于，**MySQL数据库服务器会占用多大的内存**，它们分别是：
  - 开发计算机（Development Computer）：MySQL 数据库服务器会占用所需最小的内存，以便其他应用可以正常运行。
  - 服务器计算机（System Computer）：是假设这台计算机上有多个 MySQL  数据库服务器实例在运行，因此会占用中等程度的内存。
  - 专属计算机（Dedicated Computer）：会占用计算机的全部内存资源

#### MySQL数据库的连接方式配置

- 我们也有三个选项：
  - **网络通讯协议（TCP/IP）**
  - **命名管道（Named Pipe）**
  - **内存共享（Shared Memory）**
- 命名管道和共享内存的优势是速度很快，但是它们都有一个局限，那就是只能从本机访问 MySQL  数据库服务器
- 所以我们选择 **默认的网络通讯协议方式**，这样MySQL数据库服务就可以通过网络进行访问了
  - MySQL默认的 TCP/IP 协议访问端口是 3306，后面的 X 协议端口默认是 33060
  - MySQL的 X 插件会用到 X 协议，主要用来实现类似 MongoDB 的文件存储服务

### 第三步：身份验证配置

- 关于 MySQL 的身份验证方式，我们选择系统推荐的基于 SHA256 的新加密算法 caching_sha2_password
- 跟老版本的加密算法相比，新的加密算法具有相同密码也不会生成相同的加密结果的特点

### 其余步骤省略

- 第四步：设置密码和用户权限
- 第五步：配置 Windows 服务



## 01. 存储：一个完整的数据存储过程是怎样的

- 在 MySQL 中，**一个完整的数据存储过程共有4步，分别是：创建数据库、确认字段、创建数据表、插入数据**

### 创建数据库

- **数据存储的第一步，就是创建数据库**

#### 为什么要先创建一个数据库，而不是直接创建数据表？

- 因为，从系统架构的层次上看，MySQL数据库系统从大到小依次是：
  - 数据库服务器
  - 数据库
  - 数据表
  - 数据表的行与列
- 安装程序已经帮我们安装了 MySQL  数据库服务器，所以，我们必须从创建数据库开始。
- **数据库是 MySQL 里最大的存储单元**。
  - 数据表、数据表里的数据、表与表之间的关系，还有在它们基础上衍生出来的各种工具，都存储在数据库里面。
  - **没有数据库，数据表就没有载体，也就无法存储数据**

#### 1. 如何创建数据库

```sql
-- demo是示例库名
create database demo;
```

#### 2. 如何查看数据库

```sql
show databases;
```

```sql
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| demo               |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

- `demo` 是我们通过 SQL 语句创建的数据库
- `infomation_schema`：是MySQL系统自带的数据库，主要保存 MySQL  数据库服务器的系统信息，比如数据库的名称、数据表的名称、字段名称、存取权限、数据文件所在的文件夹和系统使用的文件夹，等等
- `performance_schema`：是MySQL系统自带的数据库，可以用来监控MySQL的各类性能指标。
- `sys`：是MySQL系统自带的数据库，主要作用是：以一种更容易被理解的方式展示 MySQL 数据库服务器的各类性能指标，帮助系统管理员和开发人员监控MySQL的技术性能
- `mysql`：数据库保存了 MySQL  数据库服务器运行时需要的系统信息，比如数据文件夹、当前使用的字符集、约束检查信息，等等

- [MySQL 数据库系统的相关信息 - 官方文档](https://dev.mysql.com/doc/refman/8.0/en/system-schema.html)

### 确认字段

- 略

### 创建数据表

- **数据表是用来存储数据的最主要工具**

```sql
CREATE TABLE demo.test 
(
 barcode text,
 goodsname text,
 price int
);
```

#### 如何查看表的结构

```sql
DESCRIBE demo.test;
```

- 运行结果如下：

```sql
mysql> DESCRIBE demo.test;
+-----------+------+------+-----+---------+-------+
| Field     | Type | Null | Key | Default | Extra |
+-----------+------+------+-----+---------+-------+
| barcode   | text | YES  |     | NULL    |       |
| goodsname | text | YES  |     | NULL    |       |
| price     | int  | YES  |     | NULL    |       |
+-----------+------+------+-----+---------+-------+
3 rows in set (0.00 sec)
```

- Field：表示字段名称
- Type：表示字段类型
- Null：表示该字段是否允许是空值（NULL），注意：MySQL中，空值不等于空字符串，一个空字符串的长度是0，而一个空值的长度是空。在MySQL里，空值是占用空间的。
- Key：键
- Default：表示默认值
- Extra：附加信息

#### 如何查看数据库中的表

```sql
SHOW TABLES;
```

```sql
mysql> SHOW TABLES;
+----------------+
| Tables_in_demo |
+----------------+
| test           |
+----------------+
1 row in set (0.00 sec)
```

#### 如何设置主键

- **主键可以确保数据的唯一性，而且能够减少数据错误**
  - 必须唯一，不能重复
  - 不能是空
  - 必须可以唯一标识数据表中的记录
  - 一个 MySQL 数据表中只能有一个主键

```sql
alter table demo.test
add column itemnumber int PRIMARY KEY AUTO_INCREMENT;
```

- alter table：表示修改表
- add column：表示增加一列
- primary key：表示是主键
- auto_increment：自增

### 插入数据

```sql
INSERT INTO demo.test (barcode,goodsname,price) VALUES ('0001','本',3);
```

- insert into 表示向 demo.test 中插入数据，后面是要插入数据的字段名，values 表示对应的值

### SQL书写规范

[MySQL书写规范文档](https://www.sqlstyle.guide/zh/)

### 精选问答

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679644319130.png)

- 这里回答应该是假设 sex 非空，每行数据都必有 主键 和 sex。
- Innodb 中，主键索引所在的 B+ Tree，树的每个非叶子节点是主键，叶子节点是整行数据（比如10个字段）
- 而 sex 索引中，每个非叶子节点是 sex 的值，sex 可选值最多就 男、女、未知，指向的叶子节点聚合起来就是所有的主键，而全部主键的数量就是全部数据的数量。



## 02. 字段：这么多字段类型，该怎么定义？

- MySQL 中有很多字段类型，比如整数、文本、浮点数，等等...
- 如果类型定义合理，就能节省存储空间，提升数据查询和处理的速度，相反，如果数据类型定义不合理，就可能导致数据超出取值范围，引发系统报错，甚至可能出现计算错误的情况，进而影响到整个系统。

### 整数类型

- 整数类型一共有5种，包括 **tinyint、smallint、mediumint、int 和 bigint**，它们区别如下：

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679646647121.png)

### 浮点数类型和定点数类型

- 浮点数和定点数类型的特点是可以处理小数，可以把整数看成小数的一个特例。

- MySQL 支持的浮点数类型：

  - `FLOAT`：表示单精度浮点数

    - 占用字节数少，取值范围小

  - `DOUBLE`：表示双精度浮点数

    - 占用字节数多，取值范围大

  - `REAL`：默认就是D double。如果把 sql 模式设定为启用 `REAL_AS_FLOAT`，那么 MySQL 就认为 real 是 float。

    - ```sql
      SET sql_mode = “REAL_AS_FLOAT”;
      ```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679646930812.png)

- 问题：为什么浮点数类型的无符号数取值范围，只相当于有符号数取值范围的一半，也就是只相当于有符号数取值范围大于等于零的部分呢？
- 原因：MySQL 是按照这个格式存储浮点数的：
  - 符号（S）、尾数（M）和阶码（E）。
  - 因此，无论有没有符号，MySQL 的浮点数都会存储表示符号的部分。
  - 因此，所谓的无符号数取值范围，其实就是有符号数取值范围大于等于零的部分。
- **浮点数类型有个缺陷，就是不精确**，因此，在一些对精确度要求较高的项目中，千万不要使用浮点数！

#### 为什么MySQL的浮点数不够精准

- 举一个实际的例子：

```sql
CREATE TABLE demo.goodsmaster
(
  barcode TEXT,
  goodsname TEXT,
  price DOUBLE,
  itemnumber INT PRIMARY KEY AUTO_INCREMENT
);
```

```sql
-- 第一条
INSERT INTO demo.goodsmaster
(
  barcode,
  goodsname,
  price
)
VALUES
(
  '0001',
  '书',
  0.47
),
(
  '0002',
  '笔',
  0.44
),
(
  '0002',
  '胶水',
  0.19
);
```

```sql
mysql> SELECT * FROM demo.goodsmaster;
+---------+-----------+-------+------------+
| barcode | goodsname | price | itemnumber |
+---------+-----------+-------+------------+
| 0001    | 书        |  0.47 |          1 |
| 0002    | 笔        |  0.44 |          2 |
| 0002    | 胶水      |  0.19 |          3 |
+---------+-----------+-------+------------+
3 rows in set (0.00 sec)
```

```sql
SELECT SUM(price) FROM demo.goodsmaster;
```

- SUM，MySQL中的求和函数，是MySQL聚合函数的一种
- 期待的结果应该是 0.47 + 0.44 + 0.19 = 1.1，可是得到的是..

```sql
mysql> SELECT SUM(price)
    -> FROM demo.goodsmaster;
+--------------------+
| SUM(price)         |
+--------------------+
| 1.0999999999999999 |
+--------------------+
```

- 这就是误差，改成 float，得到的是 1.0999999940395355，误差更大



- 为什么会存在这样的误差？**问题出在MySQL对浮点类型数据的存储方式上**
  - MySQL用4个字节存储 float 类型数据，用8个字节存储 double 类型数据，
  - 无论哪个，都是采用二进制的方式来进行存储的。
  - 比如 9.625，用二进制来表达，就是 1001.101，或者表达成 1.00101 x 2 ^ 3
  - 如果尾数不是0或5，就无法用一个二进制数来精确表达，那就只好在取值允许的范围内进行近似（四舍五入）
- 为什么 double 误差比 float 小呢？原因就是 double 有8字节，精度更高



#### MySQL精准的数据类型：定点数类型 | decimal

- 就像浮点数类型的存储方式，决定了它不可能精准一样，decimal 的存储方式决定了它一定是精准的。
- decimal 是把十进制数的整数部分和小数部分拆开，分别转换成 十六进制 数，进行存储。
  - 这样，所有的数值，就都可以精准表达了，不会存在因为无法表达而丢失精度的问题
- MySQL 用 decimal（M,D）的方式表示高精度小数。
  - M表示整数加小数部分，一共有多少位，M <= 65
  - D表示小数部分位数，D < M



- 修改上面的字段类型验证一下

```sql
ALTER TABLE demo.goodsmaster MODIFY COLUMN price DECIMAL(5,2);
```

```sql
SELECT SUM(price) FROM demo.goodsmaster;
```

- 这次的结果是正确的，1.10



### 文本类型

- 即 字符串数据，MySQL支持的文本类型如下：
  - char(M)：固定长度字符串。必须预先定义字符串长度，如果太短，数据可能会超出范围，如果太长，又浪费存储空间。
  - varchar(M)：可变长度字符串。也需要预先知道字符串的最大长度，不过只要不超过这个最大长度，具体存储的时候，是按照实际字符串长度存储的。
  - text：字符串。系统自动按照实际长度存储，不需要预先定义长度。
  - enum：枚举类型。取值必须是预先设定的一组字符串值范围之内的一个，必须要知道字符串所有可能的取值。
  - set：是一个字符串对象。取值必须是在预先设定的字符串值范围之内的0个或多个，也必须知道字符串所有可能的取值。
- 因为 text 最灵活方便，不需要预先知道字符串的长度，系统会按照实际的数据长度进行存储，所以重点学习一下。
  - text 类型也有4种，它们的区别就是最大长度不同：
    - tinytext：255字符（这里假设字符是 ascll 码，一个字符占用一个字节，下同）
    - text：65535 字符
    - mediumtext：16777215 字符。
    - longtext：4294967295 字符(相当于 4GB)。
  - 注意：text 也有一个问题，**由于实际存储的长度不确定，MySQL不允许text类型的字段做主键**
    - 遇到需要用字符串做主键的情况，只能使用 char(M) 或 varchar(M)
- 所以建议：只要不是主键字段，就可以按照数据可能的最大长度，选择这几种text类型中的一种，作为存储字符串的数据类型。



### 日期与时间类型

- **用得最多的日期时间类型，就是 datetime**
- 虽然 MySQL 也支持 year(年)、date(日期)、time(时间) 以及 timestamp 类型，但是 **建议在实际项目中，尽量用 datetime 类型**。
  - 因为这个类型包括了完整的日期和时间信息，使用起来比较方便。

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1679649341282.png)

- 为什么时间类型 time 的取值范围不是 -23:59:59~23:59:59 呢?
  - 原因是 MySQL 设计的 time 类型，不光表示一天之内的时间，而且可以用来表示一个时间间隔，这个时间间隔可以超过24小时

### 总结

- 定义数据类型时，推荐的是：
  - 整数：int
  - 小数：decimal
  - 字符串：text
  - 日期与时间：datetime
- 这样做的好处是，首先确保系统不会因为数据类型定义出错。
- 不过，可靠性好，并不意味着高效。比如 text 虽然使用方便，但是效率不如 char(M) 和 varchar(M)
- 如果有进一步优化的需求，可以参考 [官方文档](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)



## 03. 表：怎么创建和修改数据表？

