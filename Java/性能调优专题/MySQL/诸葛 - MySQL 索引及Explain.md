[TOC]

#   诸葛 - MySQL 索引及Explain

## 索引的本质

```shell
# 索引是帮助 MySQL 高效获取数据的排好序的数据结构。
```

## 索引的数据结构

```shell
# 常用的数据结构:
	# 二叉树
	# 红黑树
	# Hash表
	# B-Tree
```

## 索引的重要性分析

![UTOOLS1584427860048.png](http://yanxuan.nosdn.127.net/9b74c983233ebd4d6fb37c51a47fe500.png)

```sql
-- 假设我们要查询其中一条数据:
-- 假设该表中没有任何索引，那么就会对表进行全盘扫描
-- 那么发生磁盘 IO 的次数和表中数据个数 基本一致，千万级数据表的查询性能就会极其低下。
select * from t where t.col2 = 89
```

## MySQL 索引数据结构选型分析

### 动态数据结构插入动画演示网站

```shell
# https://www.cs.usfca.edu/~galles/visualization/Algorithms.html
```

### 二叉树

![UTOOLS1584427800305.png](http://yanxuan.nosdn.127.net/8b74f70d12ec07a6e5a75f33a9147f95.png)

```sql
-- 假设我们使用二叉树作为 MySQL 索引的存储数据结构:
-- 同样的查询条件，由于 col2 字段使用二叉树索引
-- 利用二分搜索法可以提高极大查询效率，减少磁盘 IO 次数
-- 比如同样的 sql 语句，此时只需要查询 2 次，而上面无任何索引需要查询 6 次。
-- 二叉树的节点是 KV 键值对，K 是字段值，V 是行记录所在的磁盘偏移量位置。
select * from t where t.col2 = 89
```

![UTOOLS1584428407283.png](http://yanxuan.nosdn.127.net/1b58ebfdec30bcd5bf50aefdcccc2d48.png)

```sql
-- 二叉树的弊端:
-- 在极端情况下，二叉树退化为链表，如图所示。
-- 此时二叉树的索引数据和无任何索引结构，对 sql 查询的效果是一致的。
-- 同样的 sql，此时又需要发生 6 次磁盘 IO 才能查到对应数据。
select * from t where t.col2 = 89
```

### 红黑树

![UTOOLS1584429359755.png](http://yanxuan.nosdn.127.net/85eefe2184ce23a809904f5bc2a019f5.png)

```sql
-- 假设现在 MySQL 使用红黑树作为索引的数据结构
-- 由于红黑树的特性（子树之间的高度差不能大于2）会不断自旋调整树的高度，
-- 所以红黑树是不会退化为链表的
-- 同样的 SQL 此时产生的 IO 次数为 3 次
select * from t where t.col2 = 89
```

```shell
# 红黑树的弊端:

# 1. 假如单表数据达到千万级，红黑树的高度也会变得很高，磁盘 IO 的次数也会增长起来。
	# 单表五百万的数据，红黑树的高度就可能达到 20 左右，每次查询的磁盘IO 20次,是难以忍受的.

# 2. 每次插入数据可能产生的自旋，会使数据的插入速度变慢。
```

### B-Tree

![UTOOLS1584430193554.png](http://yanxuan.nosdn.127.net/c7a170ca8603193ad9ccd34d3aeadfe5.png)

```sql
-- 假设现在使用 B-Tree 作为索引的数据结构
-- 在单个节点中存储更多的数据，提高横向存储能力，减少纵向树的高度
-- 以达到减少磁盘 IO 次数，提高查询效率
-- 同样的 SQL 此时产生的磁盘 IO 次数为 2，甚至可能是1（影响因素: 单节点的索引数据容量）
select * from t where t.col2 = 89
```

```shell
# 补充: B-Tree 的特点
	# 叶子节点都具有相同的深度
	
	# 节点中的数据索引从左到右递增排列
```

![UTOOLS1584430889672.png](http://yanxuan.nosdn.127.net/bb98d4f9a76f8c588f16cdc0a6af6f6e.png)

```shell
# B-Tree 的弊端:

# SHOW GLOBAL STATUS LIKE 'Innodb_page_size';

# 1. B-Tree 中单个索引元素中还是以 KV 形式存储，导致单个索引元素占用了更多的空间
	# 由于 MySQL 对单个索引节点的空间设定大概是 16kb，物理空间固定。
	# 就是影响了横向扩展，使每次加载到内存中的索引元素变少，提高了查询的磁盘 IO 次数。
	# MyISAM 存储引擎中，data 里存放的是数据行所在的磁盘物理地址。
	# Innodb 存储引擎中，data 里存放的可能就是数据行中其他所有列的数据了。
	
# 2. B-Tree 无法实现区间访问的快速性
	# 比如: select * from t where t.col2 > 22
```

### Hash

![UTOOLS1583808278259.png](http://yanxuan.nosdn.127.net/8986d2c44c07d6b273ac76c5edebeed8.png)

```shell
# MySQL 中提供了 Hash 和 B+Tree 两种索引存储结构

# 其中 Hash 算法在 = 条件时查询效率甚至比 B+Tree 更高

# 但是 Hash 的痛点还是在于无法实现区间查询的快速性
```

### B+Tree

![UTOOLS1583303788937.png](http://yanxuan.nosdn.127.net/b03923ad19def93e6f671995c51f4cbe.png)

```shell
# MySQL 最终对 B-Tree 进行改良得到了 B+Tree

# B+Tree 的特点:
	# 非叶子节点不存储 data 数据，为了单页能存储更多的索引元素，减少树的高度。
	
	# 对索引元素进行冗余。
	
	# 叶子节点利用指针连接，大大提高了区间访问的性能。
	
# 根据 MySQL 的设定和算法，B+Tree 树的高度达到 3 时，差不多就能存 2000 多万条数据。

# 所以在千万级数据表中，如果不对查询条件的字段建立索引，
	# 就是千万次磁盘 IO 和个位数磁盘 IO 的对比。
```

## MySQL 的物理数据目录

```shell
# MySQL 的数据存储目录在安装目录下的 data 目录中

# 每个数据库对应单独一个文件夹
```

### MyISAM 的存储格式

![UTOOLS1583305533054.png](http://yanxuan.nosdn.127.net/5d08e1375a69058145329f3735f606e0.png)

```shell
# MyISAM 索引文件和数据文件是分离的（非聚集索引）
	# 数据在 MYD 文件中
	# 索引在 MYI 文件中
	# 表的结构信息在 frm 文件中
```

### InnoDB 的存储格式

![UTOOLS1583308082734.png](http://yanxuan.nosdn.127.net/a8772efb0aba434117f22720d02c6e57.png)

```shell
# InnoDB 引擎的表只有两个文件
	# frm : 存储表的结构信息
	
	# ibd : 存储表的数据和索引
	
# InnoDB 索引实现(聚集)
	# 表数据文件本身就是按 B+Tree 组织的一个索引结构文件
	# 聚集索引-叶子节点包含了完整的数据记录
	
# 为什么 InnoDB 表必须有主键，并且推荐使用整型的自增主键？
	# 在主键索引结构中，叶子节点的 data 数据即完整的数据行记录
	# 聚集索引就是: 叶子节点中 data 为完整的数据行记录
	# 推荐整型: 是因为 B+Tree 中索引的比较大小方便，利于二分查找
		# 从存储空间来说，整型较小，一次可以加载更多的主键数据
		
	# 推荐自增: 往索引结构中添加数据时便于区间排序，
		# 契合 B+Tree 叶子节点的区间指针，利于快速区间范围查找

# 为什么非主键索引结构叶子节点存储的是主键值？(一致性和节省存储空间)
	# 因为 InnoDB 中数据和索引是在同一个文件中
	# 非主键索引的叶子节点 data 指向主键
	# 再由找到的主键去主键索引中找到最终对应的数据行记录
```

## 联合索引的底层存储结构

```shell
# 在实际工作中，一张表的字段数量通常能达到几十个。

# 对需要建立索引的字段，都建立单值索引的话，会有多个 B+Tree 需要维护，
	# 占用了更多的存储空间，以及降低了查询的效率。
	
# 例如: 
	# 一张表 20 个字段，需要对 5 个字段建立索引优化查询，
	# 如果建立 5 个单值索引，需要维护 5 个 B+Tree 结构，
	# 而如果建立联合索引，只需要维护一份。
```

![UTOOLS1584433804101.png](http://yanxuan.nosdn.127.net/1722074005e5bdb5dfa82a562f62ccd7.png)

```shell
# 联合索引:
	# 其实就是在 K 中存放了多个字段的值。
	
# 如上图所示:
	# 这是一个主键索引 B+Tree ，叶子节点存储的整个数据行的数据，而不是主键
	# (a,b,c) 联合索引可能对应的就是 (工号、部门、入职日期)
	
# 联合索引中是如何维护区间性？
	# 逐个字段比较大小，
	# 先从第一个字段开始比较，数值大的就排后面，
	# 如果第一个字段相同，就比较第二个字段，数值大的就排后面，
	# 依次类推
```

## Explain 详解与索引最佳实践

### Explain 工具介绍

```shell
# 一个帮助查看 SQL 是否走索引的工具。
```

### 示例 SQL

```sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for actor
-- ----------------------------
DROP TABLE IF EXISTS `actor`;
CREATE TABLE `actor` (
  `id` int(11) NOT NULL,
  `name` varchar(45) DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of actor
-- ----------------------------
BEGIN;
INSERT INTO `actor` VALUES (1, 'a', '2020-03-17 17:01:32');
INSERT INTO `actor` VALUES (2, 'b', '2020-03-17 17:01:38');
INSERT INTO `actor` VALUES (3, 'c', '2020-03-17 17:01:41');
COMMIT;

-- ----------------------------
-- Table structure for film
-- ----------------------------
DROP TABLE IF EXISTS `film`;
CREATE TABLE `film` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of film
-- ----------------------------
BEGIN;
INSERT INTO `film` VALUES (3, 'film0');
INSERT INTO `film` VALUES (1, 'film1');
INSERT INTO `film` VALUES (2, 'film2');
COMMIT;

-- ----------------------------
-- Table structure for film_actor
-- ----------------------------
DROP TABLE IF EXISTS `film_actor`;
CREATE TABLE `film_actor` (
  `id` int(11) NOT NULL,
  `film_id` int(11) NOT NULL,
  `actor_id` int(11) NOT NULL,
  `remark` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_film_actor_id` (`film_id`,`actor_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of film_actor
-- ----------------------------
BEGIN;
INSERT INTO `film_actor` VALUES (1, 1, 1, NULL);
INSERT INTO `film_actor` VALUES (2, 1, 2, NULL);
INSERT INTO `film_actor` VALUES (3, 2, 1, NULL);
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;
```

### Explain 使用示例

```sql
EXPLAIN SELECT * FROM actor;
```

![UTOOLS1584436180946.png](http://yanxuan.nosdn.127.net/baca56471f7f0bbc50b3d6997d9fc0c4.png)

### Explain 结果字段

```shell
# 核心的四个字段:
	# select_type
	# type
	# key
	# Extra
```

#### 演示SQL

```sql
-- 关闭 MySQL5.7 新特性对衍生表的合并优化
SET SESSION optimizer_switch = 'derived_merge=off';
EXPLAIN SELECT
	( SELECT 1 FROM actor WHERE id = 1 ) 
FROM
	( SELECT * FROM film WHERE id = 1 ) der;
```

![UTOOLS1584437051896.png](http://yanxuan.nosdn.127.net/beda3c5e6ee908e13a960ea0f23db48d.png)

#### id

```shell
# select 的序列号，有几个 select 就有几个 id，

# 并且 id 的顺序是按 select 出现的顺序增长的，

# id 列越大执行优先级越高，id 相同则从上往下执行，id 为 null 最后执行。

# 如演示 SQL 结果所示，id 为1 的 select 是第一个出现的，依次类推。
```

#### select_type

```shell
# 表示对应行是简单还是复杂的查询。

# 1. simple
	# 简单查询，查询不包含 子查询 和 union
	# 上述的 Explain 使用示例结果就是 simple 简单查询。
	
# 2. primary
	# 复杂查询中最外层的 select
	# 如演示 SQL 结果所示，id 为1 的 select 的 select_type 结果就是 primary。
	
# 3. subquery
	# 包含在 select 中的子查询（不在 from 子句中）
	# 如演示 SQL 结果所示，id 为2 的 select 的 select_type 结果就是 subquery。
	
# 4. derived
	# 包含在 from 子句中的子查询，
	# MySQL 会将结果存放在一个临时表中，也称为派生表
	# 如演示 SQL 结果所示，id 为3 的 select 的 select_type 结果就是 derived。
	
# 5. union
	# 在 union 中的第二个和随后的 select
	# 示例: explain select 1 union all select 1;
```

#### table

```shell
# explain 行正在访问的数据表。

# 当 from 子句中有子查询时，table 列是 <derivenN> 格式，
	# 表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查询。
	
# 当有 union 时，union result 的 table 列的值为<union1,2>，
	# 1 和 2 表示参与 union 的 select 行 id。
```

#### type

```shell
# 关联类型或访问类型，
	# 即 MySQL 决定如何查找表中的行，查找数据行记录的大概范围。
	
# 依次从最优到最差分为别:
	# system > const > eq_ref > ref > range > index > ALL
	
# 一般来说，得保证查询达到 range 级别，最好达到 ref
```

#### possible_keys

```shell
# 显示查询可能使用哪些索引来查找。

# explain 有时可能出现 possible_keys 有值，而 key 显示 NULL 的情况，
	# 可能是因为表中数据不多，mysql 认为索引对此查询帮助不大，所以直接选择全表扫描。
	
# 如果该列时 NULL，可以通过检查 where 子句看是否可以创建一个适当的索引来提高查询性能，
	# 然后再用 explain 查看效果。
```

#### key

```shell
# 显示 mysql 实际采用哪个索引来优化该 SQL。

# NULL 表示没有使用索引。

# 可以使用 force index 强制指定 possible_key 中的索引，
	# 或者使用 ignore index 忽略。
```

#### key_len

```shell
# 显示 sql 在索引里使用的字节数，通过该值，可以算出具体使用了索引中的哪些列。
```

```sql
-- 该 sql 使用了联合索引 idx_flim_actor_id 中的 film_id
-- 因为每个 int 是4个字节，key_len 是 4，表示只使用到了第一个索引列
explain select * from film_actor where film_id = 2;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598514542340.png)

```shell
# key_len 计算规则:
	# 字符串:
		# char(n): n字节长度
		
		# varchar(n): 
			# utf-8: 3n + 2 个字节长度，加的 2 字节用来存储字符串长度
			
			# utf8mb4: 4n + 2 个字节长度
			
	# 数值类型:
		# tinyint: 1字节
		
		# smallint: 2字节
		
		# int: 4字节
		
		# bigint: 8字节
		
	# 时间类型:
		# date: 3字节
		
		# timestamp: 4字节
		
		# datetime: 8字节
		
	# 如果字段允许为 NULL，需要 1 字节记录是否为 NULL
	
# 索引最大长度时 768 字节，当字符串过长时，
	# MySQL 会做一个类似左前缀索引的处理，
	
	# 将前半部分的字符提取出来做索引。
```

#### ref

```shell
# 显示在 key 列记录的索引中，sql 查找数据所用到的列或常量。
	# 常见的有: const(常量)、字段名(film.id)
```

#### rows

```shell
# sql 估计要读取并检测的行数，并不是结果集里的行数，也不一定准确。
```

#### extra

```shell
# 显示额外信息，常见重要值如下:
```

##### Using index

```shell
# 使用覆盖索引。

# 覆盖索引定义:
	# sql 执行计划 explain 结果里的 key 有使用索引，
		# 如果 select 后面查询的字段都可以从这个索引的树中获取，
	
		# 这种情况一般就可以说是用到了覆盖索引，
	
		# extra 里一般都有 using index。
	
	# 覆盖索引一般针对的是辅助索引，
		# 整个查询结果只通过辅助索引，就能拿到结果，
		
		# 不需要再回表到主键索引树中，获取其他字段值
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598515572662.png)

##### Using where

```shell
# 使用 where 语句来处理结果，并且查询的列未被索引覆盖。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598515705256.png)

##### Using index condition

```shell
# 查询的列不完全被索引覆盖，where 条件中是一个前导列的范围。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598515764603.png)

##### Using temporary

```shell
# MySQL 需要创建一张临时表来处理查询，
	# 这种情况一般是需要优化的，首先是想到用索引来优化。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598515855377.png)

##### Using filesort

```shell
# 使用文件排序，
	# 文件较小时，加载到内存中进行排序，否则需要在磁盘完成排序，
	
	# 这种情况一般是需要优化的，首先是想到用索引来优化。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598515982832.png)

##### Select tables optimized away

```shell
# 使用某些聚合函数（比如 max、min）来访问存在索引的某个字段是
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598516075772.png)

## 索引最佳实践

```sql
CREATE TABLE `employees` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名',
`age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄',
`position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位',
`hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
 PRIMARY KEY (`id`),
 KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 COMMENT='员工记录表';

INSERT INTO employees(name,age,position,hire_time) VALUES('LiLei',22,'manager',NOW())
INSERT INTO employees(name,age,position,hire_time) 
VALUES('HanMeimei', 23,'dev',NOW());
INSERT INTO employees(name,age,position,hire_time) 
VALUES('Lucy',23,'dev',NOW());
```

### 全值匹配

```sql
explain select * from employees where name = 'LiLei';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598516264773.png)

```sql
explain select * from employees where name = 'LiLei' and age = 22;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598516309929.png)

```sql
explain select * from employees where name = 'LiLei' and age = 22 and position = 'manager';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598516400523.png)

### 最左前缀法则

```shell
# 查询 SQL 从索引的最左前列开始，并且不跳过索引中的列。
```

```sql
-- 走 name 和 age 两个字段索引
explain select * from employees where name = 'Bill' and age = 31;

-- 不走索引，因为联合索引树是按序排列，第一个字段无法确认，自然无法确定下面字段的顺序。
explain select * from employees where age = 30 and position = 'dev';

-- 不走索引，同理
explain select * from employees where position = 'manager';
```

### 不在索引列上做任何操作（计算、函数、自动或手动类型转换）

```sql
-- 不走索引
explain select * from employees where left(name,3) = 'LiLei';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598518769951.png)

### 范围查询条件后的条件不走索引

```sql
-- position 不走索引
explain select * from employees where name = 'LiLei' and age > 22 and position = 'manager';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598518964036.png)

### 尽量使用覆盖索引

```sql
explain select * from employees where name = 'LiLei' and age = 23 and position = "manager";
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598519079953.png)

### 范围条件中使用索引的规则

```shell
# mysql 在使用不等于 (!= 或 <>)、not in、not exists 时，无法使用索引。

# < 、> 、<= 、>=
	# mysql 内部优化器会根据检索比例、表的大小等多个因素整体评估是否使用索引。
```

```sql
explain select * from employees where name != 'LiLei';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598520460423.png)

### is null，is not null

```shell
# 一般情况下都不会走索引
```

```sql
explain select * from employees where name is null;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598520586437.png)

### 模糊查询

```shell
# like 以通配符 % 开头，索引失效
```

```sql
explain select * from employees where name like '%Lei';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598520821413.png)

### 字符串不加单引号索引失效

```sql
explain select * from employees where name = 1000;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598521336546.png)

### 少用 or 或 in

```shell
# 用 or 或 in 查询时，mysql 内部优化器会根据检索比例、表大小 决定是否使用索引
```

```sql
explain select * from employees where name = 'LiLei' or name = 'HanMeimei';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598521506559.png)

### 索引使用总结

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1598521706878.png)

## ONLY_FULL_GROUP_BY

```sql
-- 5.7 关闭 only_full_group_by 报错
set sql_mode = (select replace(@@sql_mode, 'ONLY_FULL_GROUP_BY',''));
```

