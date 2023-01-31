[TOC]

# 博客园 - MySQL之JSON

# 0. 原文地址

[原文地址](https://www.cnblogs.com/ivictor/p/16221712.html)

JSON 数据类型是 MySQL 5.7.8 开始支持的。在此之前，只能通过字符类型（CHAR，VARCHAR 或 TEXT ）来保存 JSON 文档。

相对字符类型，原生的 JSON 类型具有以下优势：

1. 在插入时能自动校验文档是否满足 JSON 格式的要求。
2. 优化了存储格式。无需读取整个文档就能快速访问某个元素的值。

在 JSON 类型引入之前，如果我们想要获取 JSON 文档中的某个元素，必须首先读取整个 JSON 文档，然后在客户端将其转换为 JSON 对象，最后再通过对象获取指定元素的值。

下面是 Python 中的获取方式。

```makefile
import json

# JSON 字符串:
x =  '{ "name":"John", "age":30, "city":"New York"}'

# 将 JSON 字符串转换为 JSON 对象:
y = json.loads(x)

# 读取 JSON 对象中指定元素的值:
print(y["age"])
```

这种方式有两个弊端：一、消耗磁盘 IO，二、消耗网络带宽，如果 JSON 文档比较大，在高并发场景，有可能会打爆网卡。

如果使用的是 JSON 类型，相同的需求，直接使用 SQL 命令就可搞定。不仅能节省网络带宽，结合后面提到的函数索引，还能降低磁盘 IO 消耗。

```sql
mysql> create table t(c1 json);
Query OK, 0 rows affected (0.09 sec)

mysql> insert into t values('{ "name":"John", "age":30, "city":"New York"}');
Query OK, 1 row affected (0.01 sec)

mysql> select c1->"$.age" from t;
+-------------+
| c1->"$.age" |
+-------------+
| 30          |
+-------------+
1 row in set (0.00 sec)
```

本文将从以下几个方面展开：

1. 什么是 JSON。
2. JSON 字段的增删改查操作。
3. 如何对 JSON 字段创建索引。
4. 如何将存储 JSON 字符串的字符字段升级为 JSON 字段。
5. 使用 JSON 时的注意事项。
6. Partial Updates。
7. 其它 JSON 函数。

 

# 一、什么是 JSON

JSON 是 JavaScript Object Notation（JavaScript 对象表示法）的缩写，是一个轻量级的，基于文本的，跨语言的数据交换格式。易于阅读和编写。

JSON 的基本数据类型如下：

- 数值：十进制数，不能有前导 0，可以为负数或小数，还可以为 e 或 E 表示的指数。

- 字符串：字符串必须用双引号括起来。

- 布尔值：true，false。

- 数组：一个由零或多个值组成的有序序列。每个值可以为任意类型。数组使用方括号`[]` 括起来，元素之间用逗号`,`分隔。譬如，

  ```json
  [1, "abc", null, true, "10:27:06.000000", {"id": 1}]
  ```

- 对象：一个由零或者多个键值对组成的无序集合。其中键必须是字符串，值可以为任意类型。

  对象使用花括号`{}`括起来，键值对之间使用逗号`,`分隔，键与值之间用冒号`:`分隔。譬如，

  ```json
  {"db": ["mysql", "oracle"], "id": 123, "info": {"age": 20}}
  ```

- 空值：null。

 

# 二、JSON 字段的增删改查操作

下面我们看看 JSON 字段常见的增删改查操作：

### 2.1 插入操作

可直接插入 JSON 格式的字符串。

```sql
mysql> create table t(c1 json);
Query OK, 0 rows affected (0.03 sec)

mysql> insert into t values('[1, "abc", null, true, "08:45:06.000000"]');
Query OK, 1 row affected (0.01 sec)

mysql> insert into t values('{"id": 87, "name": "carrot"}');
Query OK, 1 row affected (0.01 sec)
```

也可使用函数，常用的有 JSON_ARRAY() 和 JSON_OBJECT()，前者用于构造 JSON 数组，后者用于构造 JSON 对象。如，

```lua
mysql> select json_array(1, "abc", null, true,curtime());
+--------------------------------------------+
| json_array(1, "abc", null, true,curtime()) |
+--------------------------------------------+
| [1, "abc", null, true, "10:12:25.000000"]  |
+--------------------------------------------+
1 row in set (0.01 sec)

mysql> select json_object('id', 87, 'name', 'carrot');
+-----------------------------------------+
| json_object('id', 87, 'name', 'carrot') |
+-----------------------------------------+
| {"id": 87, "name": "carrot"}            |
+-----------------------------------------+
1 row in set (0.00 sec)
```

对于 JSON 文档，KEY 名不能重复。

如果插入的值中存在重复 KEY，在 MySQL 8.0.3 之前，遵循 first duplicate key wins 原则，会保留第一个 KEY，后面的将被丢弃掉。

从 MySQL 8.0.3 开始，遵循的是 last duplicate key wins 原则，只会保留最后一个 KEY。

下面通过一个具体的示例来看看两者的区别。

MySQL 5.7.36

```lua
mysql> select json_object('key1',10,'key2',20,'key1',30);
+--------------------------------------------+
| json_object('key1',10,'key2',20,'key1',30) |
+--------------------------------------------+
| {"key1": 10, "key2": 20}                   |
+--------------------------------------------+
1 row in set (0.02 sec)
```

MySQL 8.0.27

```lua
mysql> select json_object('key1',10,'key2',20,'key1',30);
+--------------------------------------------+
| json_object('key1',10,'key2',20,'key1',30) |
+--------------------------------------------+
| {"key1": 30, "key2": 20}                   |
+--------------------------------------------+
1 row in set (0.00 sec)
```



### 2.2 查询操作

**JSON_EXTRACT(json_doc, path[, path] ...)**

其中，json_doc 是 JSON 文档，path 是路径。该函数会从 JSON 文档提取指定路径（path）的元素。如果指定 path 不存在，会返回 NULL。可指定多个 path，匹配到的多个值会以数组形式返回。

下面我们结合一些具体的示例来看看 path 及 JSON_EXTRACT 的用法。

首先我们看看数组。

数组的路径是通过下标来表示的。第一个元素的下标是 0。

```sql
mysql> select json_extract('[10, 20, [30, 40]]', '$[0]');
+--------------------------------------------+
| json_extract('[10, 20, [30, 40]]', '$[0]') |
+--------------------------------------------+
| 10                                         |
+--------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_extract('[10, 20, [30, 40]]', '$[0]', '$[1]','$[2][0]');
+--------------------------------------------------------------+
| json_extract('[10, 20, [30, 40]]', '$[0]', '$[1]','$[2][0]') |
+--------------------------------------------------------------+
| [10, 20, 30]                                                 |
+--------------------------------------------------------------+
1 row in set (0.00 sec)
```

除此之外，还可通过 `[M to N]` 获取数组的子集。

```sql
mysql> select json_extract('[10, 20, [30, 40]]', '$[0 to 1]');
+-------------------------------------------------+
| json_extract('[10, 20, [30, 40]]', '$[0 to 1]') |
+-------------------------------------------------+
| [10, 20]                                        |
+-------------------------------------------------+
1 row in set (0.00 sec)

# 这里的 last 代表最后一个元素的下标
mysql> select json_extract('[10, 20, [30, 40]]', '$[last-1 to last]');
+---------------------------------------------------------+
| json_extract('[10, 20, [30, 40]]', '$[last-1 to last]') |
+---------------------------------------------------------+
| [20, [30, 40]]                                          |
+---------------------------------------------------------+
1 row in set (0.00 sec)
```

也可通过 `[*]` 获取数组中的所有元素。

```sql
mysql> select json_extract('[10, 20, [30, 40]]', '$[*]');
+--------------------------------------------+
| json_extract('[10, 20, [30, 40]]', '$[*]') |
+--------------------------------------------+
| [10, 20, [30, 40]]                         |
+--------------------------------------------+
1 row in set (0.00 sec)
```



接下来，我们看看对象。

对象的路径是通过 KEY 来表示的。

```sql
mysql> set @j='{"a": 1, "b": [2, 3], "a c": 4}';
Query OK, 0 rows affected (0.00 sec)

# 如果 KEY 在路径表达式中不合法（譬如存在空格），则在引用这个 KEY 时，需用双引号括起来。
mysql> select json_extract(@j, '$.a'), json_extract(@j, '$."a c"'), json_extract(@j, '$.b[1]');
+-------------------------+-----------------------------+----------------------------+
| json_extract(@j, '$.a') | json_extract(@j, '$."a c"') | json_extract(@j, '$.b[1]') |
+-------------------------+-----------------------------+----------------------------+
| 1                       | 4                           | 3                          |
+-------------------------+-----------------------------+----------------------------+
1 row in set (0.00 sec)
```

除此之外，还可通过 `.*` 获取对象中的所有元素。

```haskell
mysql> select json_extract('{"a": 1, "b": [2, 3], "a c": 4}', '$.*');
+--------------------------------------------------------+
| json_extract('{"a": 1, "b": [2, 3], "a c": 4}', '$.*') |
+--------------------------------------------------------+
| [1, [2, 3], 4]                                         |
+--------------------------------------------------------+
1 row in set (0.00 sec)

# 这里的 $**.b 匹配 $.a.b 和 $.c.b
mysql> select json_extract('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b');
+---------------------------------------------------------+
| json_extract('{"a": {"b": 1}, "c": {"b": 2}}', '$**.b') |
+---------------------------------------------------------+
| [1, 2]                                                  |
+---------------------------------------------------------+
1 row in set (0.00 sec)
```



**column->path**

column->path，包括后面讲到的 column->>path，都是语法糖，在实际使用的时候都会转化为 JSON_EXTRACT。

column->path 等同于 JSON_EXTRACT(column, path) ，只能指定一个path。

```haskell
create table t(c2 json);

insert into t values('{"empno": 1001, "ename": "jack"}'), ('{"empno": 1002, "ename": "mark"}');

mysql> select c2, c2->"$.ename" from t;
+----------------------------------+---------------+
| c2                               | c2->"$.ename" |
+----------------------------------+---------------+
| {"empno": 1001, "ename": "jack"} | "jack"        |
| {"empno": 1002, "ename": "mark"} | "mark"        |
+----------------------------------+---------------+
2 rows in set (0.00 sec)

mysql> select * from t where c2->"$.empno" = 1001;
+------+----------------------------------+
| c1   | c2                               |
+------+----------------------------------+
|    1 | {"empno": 1001, "ename": "jack"} |
+------+----------------------------------+
1 row in set (0.00 sec)
```



**column->>path**

同 column->path 类似，只不过其返回的是字符串。以下三者是等价的。

- JSON_UNQUOTE( JSON_EXTRACT(column, path) )
- JSON_UNQUOTE(column -> path)
- column->>path

```sql
mysql> select c2->'$.ename',json_extract(c2, "$.ename"),json_unquote(c2->'$.ename'),c2->>'$.ename' from t;
+---------------+-----------------------------+-----------------------------+----------------+
| c2->'$.ename' | json_extract(c2, "$.ename") | json_unquote(c2->'$.ename') | c2->>'$.ename' |
+---------------+-----------------------------+-----------------------------+----------------+
| "jack"        | "jack"                      | jack                        | jack           |
| "mark"        | "mark"                      | mark                        | mark           |
+---------------+-----------------------------+-----------------------------+----------------+
2 rows in set (0.00 sec)
```



### 2.3 修改操作

**JSON_INSERT(json_doc, path, val[, path, val] ...)**

插入新值。

仅当指定位置或指定 KEY 的值不存在时，才执行插入操作。另外，如果指定的 path 是数组下标，且 json_doc 不是数组，该函数首先会将 json_doc 转化为数组，然后再插入新值。

下面我们看几个示例。

```lua
mysql> select json_insert('1','$[0]',"10");
+------------------------------+
| json_insert('1','$[0]',"10") |
+------------------------------+
| 1                            |
+------------------------------+
1 row in set (0.00 sec)

mysql> select json_insert('1','$[1]',"10");
+------------------------------+
| json_insert('1','$[1]',"10") |
+------------------------------+
| [1, "10"]                    |
+------------------------------+
1 row in set (0.01 sec)

mysql> select json_insert('["1","2"]','$[2]',"10");
+--------------------------------------+
| json_insert('["1","2"]','$[2]',"10") |
+--------------------------------------+
| ["1", "2", "10"]                     |
+--------------------------------------+
1 row in set (0.00 sec)

mysql> set @j = '{ "a": 1, "b": [2, 3]}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_insert(@j, '$.a', 10, '$.c', '[true, false]');
+----------------------------------------------------+
| json_insert(@j, '$.a', 10, '$.c', '[true, false]') |
+----------------------------------------------------+
| {"a": 1, "b": [2, 3], "c": "[true, false]"}        |
+----------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_SET(json_doc, path, val[, path, val] ...)**

插入新值，并替换已经存在的值。

换言之，如果指定位置或指定 KEY 的值不存在，会执行插入操作，如果存在，则执行更新操作。

```java
mysql> set @j = '{ "a": 1, "b": [2, 3]}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_set(@j, '$.a', 10, '$.c', '[true, false]');
+-------------------------------------------------+
| json_set(@j, '$.a', 10, '$.c', '[true, false]') |
+-------------------------------------------------+
| {"a": 10, "b": [2, 3], "c": "[true, false]"}    |
+-------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_REPLACE(json_doc, path, val[, path, val] ...)**

替换已经存在的值。

```java
mysql> set @j = '{ "a": 1, "b": [2, 3]}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_replace(@j, '$.a', 10, '$.c', '[true, false]');
+-----------------------------------------------------+
| json_replace(@j, '$.a', 10, '$.c', '[true, false]') |
+-----------------------------------------------------+
| {"a": 10, "b": [2, 3]}                              |
+-----------------------------------------------------+
1 row in set (0.00 sec)
```



### 2.4 删除操作

**JSON_REMOVE(json_doc, path[, path] ...)**

删除 JSON 文档指定位置的元素。

```java
mysql> set @j = '{ "a": 1, "b": [2, 3]}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_remove(@j, '$.a');
+------------------------+
| JSON_REMOVE(@j, '$.a') |
+------------------------+
| {"b": [2, 3]}          |
+------------------------+
1 row in set (0.00 sec)

mysql> set @j = '["a", ["b", "c"], "d", "e"]';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_remove(@j, '$[1]');
+-------------------------+
| JSON_REMOVE(@j, '$[1]') |
+-------------------------+
| ["a", "d", "e"]         |
+-------------------------+
1 row in set (0.00 sec)

mysql> select json_remove(@j, '$[1]','$[2]');
+--------------------------------+
| JSON_REMOVE(@j, '$[1]','$[2]') |
+--------------------------------+
| ["a", "d"]                     |
+--------------------------------+
1 row in set (0.00 sec)

mysql> select json_remove(@j, '$[1]','$[1]');
+--------------------------------+
| JSON_REMOVE(@j, '$[1]','$[1]') |
+--------------------------------+
| ["a", "e"]                     |
+--------------------------------+
1 row in set (0.00 sec)
```

最后一个查询，虽然两个 path 都是 '$[1]' ，但作用对象不一样，第一个 path 的作用对象是 '["a", ["b", "c"], "d", "e"]' ，第二个 path 的作用对象是删除了 '$[1]' 后的数组，即 '["a", "d", "e"]' 。

#  

# 三、如何对 JSON 字段创建索引

同 TEXT，BLOB 字段一样，JSON 字段不允许直接创建索引。

```sql
mysql> create table t(c1 json, index (c1));
ERROR 3152 (42000): JSON column 'c1' supports indexing only via generated columns on a specified JSON path.
```

即使支持，实际意义也不大，因为我们一般是基于文档中的元素进行查询，很少会基于整个  JSON 文档。

对文档中的元素进行查询，就需要用到 MySQL 5.7 引入的虚拟列及函数索引。

下面我们来看一个具体的示例。

```sql
# C2 即虚拟列
# index (c2) 对虚拟列添加索引。
create table t ( c1 json, c2 varchar(10) as (JSON_UNQUOTE(c1 -> "$.name")), index (c2) );

insert into t (c1) values  ('{"id": 1, "name": "a"}'), ('{"id": 2, "name": "b"}'), ('{"id": 3, "name": "c"}'), ('{"id": 4, "name": "d"}');

mysql> explain select * from t where c2 = 'a';
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | ref  | c2            | c2   | 43      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)

mysql> explain select * from t where c1->'$.name' = 'a';
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | t     | NULL       | ref  | c2            | c2   | 43      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

可以看到，无论是使用虚拟列，还是文档中的元素来查询，都可以利用上索引。

注意，在创建虚拟列时需指定  JSON_UNQUOTE，将 c1 -> "$.name" 的返回值转换为字符串。

#  

# 四、如何将存储 JSON 字符串的字符字段升级为 JSON 字段

在 MySQL 支持 JSON 类型之前，对于 JSON 文档，一般是以字符串的形式存储在字符类型（VARCHAR 或 TEXT）中。

在 JSON 类型出来之后，如何将这些字符字段升级为 JSON 字段呢？

为方便演示，这里首先构建测试数据。

```verilog
create table t (id int auto_increment primary key, c1 text);

insert into t (c1) values ('{"id": "1", "name": "a"}'), ('{"id": "2", "name": "b"}'), ('{"id": "3", "name": "c"}'), ('{"id", "name": "d"}');
```

注意，最后一个文档有问题，不是合格的 JSON 文档。

如果使用 DDL 直接修改字段的数据类型，会报错。

```sql
mysql> alter table t modify c1 json;
ERROR 3140 (22032): Invalid JSON text: "Missing a colon after a name of object member." at position 5 in value for column '#sql-7e1c_1f6.c1'.
```

下面，我们看看具体的升级步骤。

（1）使用 json_valid 函数找出不满足 JSON 格式要求的文档。

```csharp
mysql> select * from t where json_valid(c1) = 0;
+----+---------------------+
| id | c1                  |
+----+---------------------+
|  4 | {"id", "name": "d"} |
+----+---------------------+
1 row in set (0.00 sec)
```

（2）处理不满足 JSON 格式要求的文档。

```sql
mysql> update t set c1='{"id": "4", "name": "d"}' where id=4;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

（3）将 TEXT 字段修改为 JSON 字段。

```sql
mysql> select * from t where json_valid(c1) = 0;
Empty set (0.00 sec)

mysql> alter table t modify c1 json;
Query OK, 4 rows affected (0.13 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

#  

# 五、使用 JSON 时的注意事项

对于 JSON 类型，有以下几点需要注意：

1. 在 MySQL 8.0.13 之前，不允许对 BLOB，TEXT，GEOMETRY，JSON 字段设置默认值。从 MySQL 8.0.13 开始，取消了这个限制。

   设置时，注意默认值需通过小括号`()`括起来，否则的话，还是会提示 JSON 字段不允许设置默认值。

   ```sql
   mysql> create table t(c1 json not null default (''));
   Query OK, 0 rows affected (0.03 sec)
   
   mysql> create table t(c1 json not null default '');
   ERROR 1101 (42000): BLOB, TEXT, GEOMETRY or JSON column 'c1' can't have a default value
   ```

2. 不允许直接创建索引，可创建函数索引。

3. JSON 列的最大大小和 LONGBLOB（LONGTEXT）一样，都是 4G。

4. 插入时，单个文档的大小受到 max_allowed_packet 的限制，该参数最大是 1G。

#  

# 六、Partial Updates

在 MySQL 5.7 中，对 JSON 文档进行更新，其处理策略是，删除旧的文档，再插入新的文档。即使这个修改很微小，只涉及几个字节，也会替换掉整个文档。很显然，这种处理方式的效率较为低下。

在 MySQL 8.0 中，针对 JSON 文档，引入了一项新的特性-Partial Updates（部分更新），支持 JSON 文档的原地更新。得益于这个特性，JSON 文档的处理性能得到了极大提升。

下面我们具体来看看。

### 6.1 使用 Partial Updates 的条件

为方便阐述，这里先构造测试数据。

```haskell
create table t (id int auto_increment primary key, c1 json);

insert into t (c1) values  ('{"id": 1, "name": "a"}'), ('{"id": 2, "name": "b"}'), ('{"id": 3, "name": "c"}'), ('{"id": 4, "name": "d"}');

mysql> select * from t;
+----+------------------------+
| id | c1                     |
+----+------------------------+
|  1 | {"id": 1, "name": "a"} |
|  2 | {"id": 2, "name": "b"} |
|  3 | {"id": 3, "name": "c"} |
|  4 | {"id": 4, "name": "d"} |
+----+------------------------+
4 rows in set (0.00 sec)
```

使用 Partial Updates 需满足以下条件：

1. 被更新的列是 JSON 类型。

2. 使用 JSON_SET，JSON_REPLACE，JSON_REMOVE 进行 UPDATE 操作，如，

   ```bash
   update t set c1=json_remove(c1,'$.id') where id=1;
   ```

   不使用这三个函数，而显式赋值，就不会进行部分更新，如，

   ```swift
   update t set c1='{"id": 1, "name": "a"}' where id=1;
   ```

3. 输入列和目标列必须是同一列，如，

   ```bash
   update t set c1=json_replace(c1,'$.id',10) where id=1;
   ```

   否则的话，就不会进行部分更新，如，

   ```bash
   update t set c1=json_replace(c2,'$.id',10) where id=1;
   ```

4. 变更前后，JSON 文档的空间使用不会增加。

关于最后一个条件，我们看看下面这个示例。

```csharp
mysql> select *,json_storage_size(c1),json_storage_free(c1) from t where id=1;
+----+------------------------+-----------------------+-----------------------+
| id | c1                     | json_storage_size(c1) | json_storage_free(c1) |
+----+------------------------+-----------------------+-----------------------+
|  1 | {"id": 1, "name": "a"} |                    27 |                     0 |
+----+------------------------+-----------------------+-----------------------+
1 row in set (0.00 sec)

mysql> update t set c1=json_remove(c1,'$.id') where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select *,json_storage_size(c1),json_storage_free(c1) from t where id=1;
+----+---------------+-----------------------+-----------------------+
| id | c1            | json_storage_size(c1) | json_storage_free(c1) |
+----+---------------+-----------------------+-----------------------+
|  1 | {"name": "a"} |                    27 |                     9 |
+----+---------------+-----------------------+-----------------------+
1 row in set (0.00 sec)

mysql> update t set c1=json_set(c1,'$.id',3306) where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select *,json_storage_size(c1),json_storage_free(c1) from t where id=1;
+----+---------------------------+-----------------------+-----------------------+
| id | c1                        | json_storage_size(c1) | json_storage_free(c1) |
+----+---------------------------+-----------------------+-----------------------+
|  1 | {"id": 3306, "name": "a"} |                    27 |                     0 |
+----+---------------------------+-----------------------+-----------------------+
1 row in set (0.00 sec)

mysql> update t set c1=json_set(c1,'$.id','mysql') where id=1;
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select *,json_storage_size(c1),json_storage_free(c1) from t where id=1;
+----+------------------------------+-----------------------+-----------------------+
| id | c1                           | json_storage_size(c1) | json_storage_free(c1) |
+----+------------------------------+-----------------------+-----------------------+
|  1 | {"id": "mysql", "name": "a"} |                    33 |                     0 |
+----+------------------------------+-----------------------+-----------------------+
1 row in set (0.00 sec)
```

示例中，用到了两个函数：JSON_STORAGE_SIZE 和 JSON_STORAGE_FREE ，前者用来获取 JSON 文档的空间使用情况，后者用来获取 JSON 文档在执行原地更新后的空间释放情况。

这里一共执行了三次 UPDATE 操作，前两次是原地更新，第三次不是。同样是 JSON_SET 操作，为什么第一次是原地更新，而第二次不是呢？

因为第一次的 JSON_SET 复用了 JSON_REMOVE 释放的空间。而第二次的 JSON_SET 执行的是更新操作，且 'mysql' 比 3306 需要更多的存储空间。



### 6.2 如何在 binlog 中开启 Partial Updates

Partial Updates 不仅仅适用于存储引擎层，还可用于主从复制场景。

主从复制开启 Partial Updates，只需将参数 binlog_row_value_options（默认为空）设置为 PARTIAL_JSON。

下面具体来看看，同一个 UPDATE 操作，开启和不开启 Partial Updates，在 binlog 中的记录有何区别。

```bash
update t set c1=json_replace(c1,'$.id',10) where id=1;
```

不开启

```shell
### UPDATE `slowtech`.`t`
### WHERE
###   @1=1
###   @2='{"id": "1", "name": "a"}'
### SET
###   @1=1
###   @2='{"id": 10, "name": "a"}'
```

开启

```shell
### UPDATE `slowtech`.`t`
### WHERE
###   @1=1
###   @2='{"id": 1, "name": "a"}'
### SET
###   @1=1
###   @2=JSON_REPLACE(@2, '$.id', 10)
```

对比 binlog 的内容，可以看到，不开启，无论是修改前的镜像（before_image）还是修改后的镜像（after_image），记录的都是完整文档。而开启后，对于修改后的镜像，记录的是命令，而不是完整文档，这样可节省近一半的空间。

在将 binlog_row_value_options 设置为 PARTIAL_JSON 后，对于可使用 Partial Updates 的操作，在 binlog 中，不再通过 ROWS_EVENT 来记录，而是新增了一个 PARTIAL_UPDATE_ROWS_EVENT 的事件类型。

需要注意的是，binlog 中使用 Partial Updates，只需满足存储引擎层使用 Partial Updates 的前三个条件，无需考虑变更前后，JSON 文档的空间使用是否会增加。

### 6.3 关于 Partial Updates 的性能测试

首先构造测试数据，t 表一共有 16 个文档，每个文档近 10 MB。

```sql
create table t(id int auto_increment primary key,
               json_col json,
               name varchar(100) as (json_col->>'$.name'),
               age int as (json_col->'$.age'));

insert into t(json_col) values
(json_object('name', 'Joe', 'age', 24,
             'data', repeat('x', 10 * 1000 * 1000))),
(json_object('name', 'Sue', 'age', 32,
             'data', repeat('y', 10 * 1000 * 1000))),
(json_object('name', 'Pete', 'age', 40,
             'data', repeat('z', 10 * 1000 * 1000))),
(json_object('name', 'Jenny', 'age', 27,
             'data', repeat('w', 10 * 1000 * 1000)));

insert into t(json_col) select json_col from t;
insert into t(json_col) select json_col from t;
```

接下来，测试下述 SQL

```sql
update t set json_col = json_set(json_col, '$.age', age + 1);
```

在以下四种场景下的执行时间：

1. MySQL 5.7.36
2. MySQL 8.0.27
3. MySQL 8.0.27，binlog_row_value_options=PARTIAL_JSON
4. MySQL 8.0.27，binlog_row_value_options=PARTIAL_JSON + binlog_row_image=MINIMAL

分别执行 10 次，去掉最大值和最小值后求平均值。

最后的测试结果如下

![img](https://img2023.cnblogs.com/blog/576154/202211/576154-20221128141825197-679136064.png)

以 MySQL 5.7.36 的查询时间作为基准：

1. MySQL 8.0 只开启存储引擎层的 Partial Updates，查询时间比 MySQL 5.7 快 1.94 倍。
2. MySQL 8.0 同时开启存储引擎层和 binlog 中的 Partial Updates，查询时间比 MySQL 5.7 快 4.87 倍。
3. 如果在 2 的基础上，同时将 binlog_row_image 设置为 MINIMAL，查询时间更是比 MySQL 5.7 快 102.22 倍。

当然，在生产环境，我们一般很少将 binlog_row_image 设置为 MINIMAL。

但即使如此，只开启存储引擎层和 binlog 中的 Partial Updates，查询时间也比 MySQL 5.7 快 4.87 倍，性能提升还是比较明显的。

#  

# 七、其它 JSON 函数

### 7.1 查询相关

**JSON_CONTAINS(target, candidate[, path])**

判断 target 文档是否包含 candidate 文档，如果包含，则返回 1，否则是 0。

```sql
mysql> set @j = '{"a": [1, 2], "b": 3, "c": {"d": 4}}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_contains(@j, '1', '$.a'),json_contains(@j, '1', '$.b');
+-------------------------------+-------------------------------+
| json_contains(@j, '1', '$.a') | json_contains(@j, '1', '$.b') |
+-------------------------------+-------------------------------+
|                             1 |                             0 |
+-------------------------------+-------------------------------+
1 row in set (0.00 sec)

mysql> select json_contains(@j,'{"d": 4}','$.a'),json_contains(@j,'{"d": 4}','$.c');
+------------------------------------+------------------------------------+
| json_contains(@j,'{"d": 4}','$.a') | json_contains(@j,'{"d": 4}','$.c') |
+------------------------------------+------------------------------------+
|                                  0 |                                  1 |
+------------------------------------+------------------------------------+
1 row in set (0.00 sec)
```



**JSON_CONTAINS_PATH(json_doc, one_or_all, path[, path] ...)**

判断指定的 path 是否存在，存在，则返回 1，否则是 0。

函数中的 one_or_all 可指定 one 或 all，one 是任意一个路径存在就返回 1，all 是所有路径都存在才返回 1。

```sql
mysql> set @j = '{"a": [1, 2], "b": 3, "c": {"d": 4}}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_contains_path(@j, 'one', '$.a', '$.e'), json_contains_path(@j, 'all', '$.a', '$.e');
+---------------------------------------------+---------------------------------------------+
| json_contains_path(@j, 'one', '$.a', '$.e') | json_contains_path(@j, 'all', '$.a', '$.e') |
+---------------------------------------------+---------------------------------------------+
|                                           1 |                                           0 |
+---------------------------------------------+---------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_contains_path(@j, 'one', '$.c.d'),json_contains_path(@j, 'one', '$.a.d');
+----------------------------------------+----------------------------------------+
| json_contains_path(@j, 'one', '$.c.d') | json_contains_path(@j, 'one', '$.a.d') |
+----------------------------------------+----------------------------------------+
|                                      1 |                                      0 |
+----------------------------------------+----------------------------------------+
1 row in set (0.00 sec)
```



**JSON_SEARCH(json_doc, one_or_all, search_str[, escape_char[, path] ...])**

返回某个字符串（search_str）在 JSON 文档中的位置，其中，

- one_or_all：匹配的次数，one 是只匹配一次，all 是匹配所有。如果匹配到多个，结果会以数组的形式返回。
- search_str：子串，支持模糊匹配：`%` 和` _` 。
- escape_char：转义符，如果该参数不填或为 NULL，则取默认转义符`\`。
- path：查找路径。

```sql
mysql> set @j = '["abc", [{"k": "10"}, "def"], {"x":"abc"}, {"y":"bcd"}]';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_search(@j, 'one', 'abc'),json_search(@j, 'all', 'abc'),json_search(@j, 'all', 'ghi');
+-------------------------------+-------------------------------+-------------------------------+
| json_search(@j, 'one', 'abc') | json_search(@j, 'all', 'abc') | json_search(@j, 'all', 'ghi') |
+-------------------------------+-------------------------------+-------------------------------+
| "$[0]"                        | ["$[0]", "$[2].x"]            | NULL                          |
+-------------------------------+-------------------------------+-------------------------------+
1 row in set (0.00 sec)

mysql> select json_search(@j, 'all', '%b%', NULL, '$[1]'), json_search(@j, 'all', '%b%', NULL, '$[3]');
+---------------------------------------------+---------------------------------------------+
| json_search(@j, 'all', '%b%', NULL, '$[1]') | json_search(@j, 'all', '%b%', NULL, '$[3]') |
+---------------------------------------------+---------------------------------------------+
| NULL                                        | "$[3].y"                                    |
+---------------------------------------------+---------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_KEYS(json_doc[, path])**

返回 JSON 文档最外层的 key，如果指定了 path，则返回该 path 对应元素最外层的 key。

```sql
mysql> select json_keys('{"a": 1, "b": {"c": 30}}');
+---------------------------------------+
| json_keys('{"a": 1, "b": {"c": 30}}') |
+---------------------------------------+
| ["a", "b"]                            |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> select json_keys('{"a": 1, "b": {"c": 30}}', '$.b');
+----------------------------------------------+
| json_keys('{"a": 1, "b": {"c": 30}}', '$.b') |
+----------------------------------------------+
| ["c"]                                        |
+----------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_VALUE(json_doc, path)**

8.0.21 引入的，从 JSON 文档提取指定路径（path）的元素。

该函数的完整语法如下：

```yaml
JSON_VALUE(json_doc, path [RETURNING type] [on_empty] [on_error])

on_empty:
    {NULL | ERROR | DEFAULT value} ON EMPTY

on_error:
    {NULL | ERROR | DEFAULT value} ON ERROR
```

其中：

- RETURNING type：返回值的类型，不指定，则默认是 VARCHAR(512)。不指定字符集，则默认是 utf8mb4，且区分大小写。
- on_empty：如果指定路径没有值，会触发 on_empty 子句， 默认是返回 NULL，也可指定 ERROR 抛出错误，或者通过 DEFAULT value 返回默认值。
- on_error：三种情况下会触发 on_error 子句：从数组或对象中提取元素时，会解析到多个值；类型转换错误，譬如将 "abc" 转换为 unsigned 类型；值被 truncate 了。默认是返回 NULL。

```sql
mysql> select json_value('{"item": "shoes", "price": "49.95"}', '$.item');
+-------------------------------------------------------------+
| json_value('{"item": "shoes", "price": "49.95"}', '$.item') |
+-------------------------------------------------------------+
| shoes                                                       |
+-------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_value('{"item": "shoes", "price": "49.95"}', '$.price' returning decimal(4,2)) as price;
+-------+
| price |
+-------+
| 49.95 |
+-------+
1 row in set (0.00 sec)

mysql> select json_value('{"item": "shoes", "price": "49.95"}', '$.price1' error on empty);
ERROR 3966 (22035): No value was found by 'json_value' on the specified path.

mysql> select json_value('[1, 2, 3]', '$[1 to 2]' error on error);
ERROR 3967 (22034): More than one value was found by 'json_value' on the specified path.

mysql> select json_value('{"item": "shoes", "price": "49.95"}', '$.item' returning unsigned error on error) as price;
ERROR 1690 (22003): UNSIGNED value is out of range in 'json_value'
```



**value MEMBER OF(json_array)**

判断 value 是否是 JSON 数组的一个元素，如果是，则返回 1，否则是 0。

```sql
mysql> select 17 member of('[23, "abc", 17, "ab", 10]');
+-------------------------------------------+
| 17 member of('[23, "abc", 17, "ab", 10]') |
+-------------------------------------------+
|                                         1 |
+-------------------------------------------+
1 row in set (0.00 sec)

mysql> select cast('[4,5]' as json) member of('[[3,4],[4,5]]');
+--------------------------------------------------+
| cast('[4,5]' as json) member of('[[3,4],[4,5]]') |
+--------------------------------------------------+
|                                                1 |
+--------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_OVERLAPS(json_doc1, json_doc2)**

MySQL 8.0.17 引入的，用来比较两个 JSON 文档是否有相同的键值对或数组元素，如果有，则返回 1，否则是 0。如果两个参数都是标量，则判断这两个标量是否相等。

```sql
mysql> select json_overlaps('[1,3,5,7]', '[2,5,7]'),json_overlaps('[1,3,5,7]', '[2,6,8]');
+---------------------------------------+---------------------------------------+
| json_overlaps('[1,3,5,7]', '[2,5,7]') | json_overlaps('[1,3,5,7]', '[2,6,8]') |
+---------------------------------------+---------------------------------------+
|                                     1 |                                     0 |
+---------------------------------------+---------------------------------------+
1 row in set (0.00 sec)

mysql> select json_overlaps('{"a":1,"b":2}', '{"c":3,"d":4,"b":2}');
+-------------------------------------------------------+
| json_overlaps('{"a":1,"b":2}', '{"c":3,"d":4,"b":2}') |
+-------------------------------------------------------+
|                                                     1 |
+-------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_overlaps('{"a":1,"b":2}', '{"c":3,"d":4,"b":10}');
+--------------------------------------------------------+
| json_overlaps('{"a":1,"b":2}', '{"c":3,"d":4,"b":10}') |
+--------------------------------------------------------+
|                                                      0 |
+--------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_overlaps('5', '5'),json_overlaps('5', '6');
+-------------------------+-------------------------+
| json_overlaps('5', '5') | json_overlaps('5', '6') |
+-------------------------+-------------------------+
|                       1 |                       0 |
+-------------------------+-------------------------+
1 row in set (0.00 sec)
```

从 MySQL 8.0.17 开始，InnoDB 支持多值索引，可用在 JSON 数组中。当我们使用 JSON_CONTAINS、MEMBER OF、JSON_OVERLAPS 进行数组相关的操作时，可使用多值索引来加快查询。



### 7.2 修改相关

**JSON_ARRAY_APPEND(json_doc, path, val[, path, val] ...)**

向数组指定位置追加元素。如果指定 path 不存在，则不添加。

```java
mysql> set @j = '["a", ["b", "c"], "d"]';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_array_append(@j, '$[0]', 1, '$[1][0]', 2, '$[3]', 3);
+-----------------------------------------------------------+
| json_array_append(@j, '$[0]', 1, '$[1][0]', 2, '$[3]', 3) |
+-----------------------------------------------------------+
| [["a", 1], [["b", 2], "c"], "d"]                          |
+-----------------------------------------------------------+
1 row in set (0.00 sec)

mysql> set @j = '{"a": 1, "b": [2, 3], "c": 4}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_array_append(@j, '$.b', 'x', '$', 'z');
+---------------------------------------------+
| json_array_append(@j, '$.b', 'x', '$', 'z') |
+---------------------------------------------+
| [{"a": 1, "b": [2, 3, "x"], "c": 4}, "z"]   |
+---------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_ARRAY_INSERT(json_doc, path, val[, path, val] ...)**

向数组指定位置插入元素。

```java
mysql> set @j = '["a", ["b", "c"],{"d":"e"}]';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_array_insert(@j, '$[0]', 1);
+----------------------------------+
| json_array_insert(@j, '$[0]', 1) |
+----------------------------------+
| [1, "a", ["b", "c"], {"d": "e"}] |
+----------------------------------+
1 row in set (0.00 sec)

mysql> select json_array_insert(@j, '$[1]', cast('[1,2]' as json));
+------------------------------------------------------+
| json_array_insert(@j, '$[1]', cast('[1,2]' as json)) |
+------------------------------------------------------+
| ["a", [1, 2], ["b", "c"], {"d": "e"}]                |
+------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_array_insert(@j, '$[5]', 2);
+----------------------------------+
| json_array_insert(@j, '$[5]', 2) |
+----------------------------------+
| ["a", ["b", "c"], {"d": "e"}, 2] |
+----------------------------------+
1 row in set (0.00 sec)
```



**JSON_MERGE_PATCH(json_doc, json_doc[, json_doc] ...)**

MySQL 8.0.3 引入的，用来合并多个 JSON 文档。其合并规则如下：

1. 如果两个文档不全是 JSON 对象，则合并后的结果是第二个文档。
2. 如果两个文档都是 JSON 对象，且不存在着同名 KEY，则合并后的文档包括两个文档的所有元素，如果存在着同名 KEY，则第二个文档的值会覆盖第一个。

```lua
mysql> select json_merge_patch('[1, 2]', '[3, 4]'), json_merge_patch('[1, 2]', '{"a": 123}');
+--------------------------------------+------------------------------------------+
| json_merge_patch('[1, 2]', '[3, 4]') | json_merge_patch('[1, 2]', '{"a": 123}') |
+--------------------------------------+------------------------------------------+
| [3, 4]                               | {"a": 123}                               |
+--------------------------------------+------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_merge_patch('{"a": 1}', '{"b": 2}'),json_merge_patch('{ "a": 1, "b":2 }','{ "a": 3, "c":4 }');
+------------------------------------------+-----------------------------------------------------------+
| json_merge_patch('{"a": 1}', '{"b": 2}') | json_merge_patch('{ "a": 1, "b":2 }','{ "a": 3, "c":4 }') |
+------------------------------------------+-----------------------------------------------------------+
| {"a": 1, "b": 2}                         | {"a": 3, "b": 2, "c": 4}                                  |
+------------------------------------------+-----------------------------------------------------------+
1 row in set (0.00 sec)

# 如果第二个文档存在 null 值，文档合并后不会输出对应的 KEY。
mysql> select json_merge_patch('{"a":1, "b":2}', '{"a":3, "b":null}');
+---------------------------------------------------------+
| json_merge_patch('{"a":1, "b":2}', '{"a":3, "b":null}') |
+---------------------------------------------------------+
| {"a": 3}                                                |
+---------------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_MERGE_PRESERVE(json_doc, json_doc[, json_doc] ...)**

MySQL 8.0.3 引入的，用来代替 JSON_MERGE。也是用来合并文档，但合并规则与 JSON_MERGE_PATCH 有所不同。

1. 两个文档中，只要有一个文档是数组，则另外一个文档会合并到该数组中。
2. 两个文档都是 JSON 对象，若存在着同名 KEY ，第二个文档并不会覆盖第一个，而是会将值 append 到第一个文档中。

```lua
mysql> select json_merge_preserve('1','2'),json_merge_preserve('[1, 2]', '[3, 4]');
+------------------------------+-----------------------------------------+
| json_merge_preserve('1','2') | json_merge_preserve('[1, 2]', '[3, 4]') |
+------------------------------+-----------------------------------------+
| [1, 2]                       | [1, 2, 3, 4]                            |
+------------------------------+-----------------------------------------+
1 row in set (0.00 sec)

mysql> select json_merge_preserve('[1, 2]', '{"a": 123}'), json_merge_preserve('{"a": 123}', '[3,4]');
+---------------------------------------------+--------------------------------------------+
| json_merge_preserve('[1, 2]', '{"a": 123}') | json_merge_preserve('{"a": 123}', '[3,4]') |
+---------------------------------------------+--------------------------------------------+
| [1, 2, {"a": 123}]                          | [{"a": 123}, 3, 4]                         |
+---------------------------------------------+--------------------------------------------+
1 row in set (0.00 sec)

mysql> select json_merge_preserve('{"a": 1}', '{"b": 2}'), json_merge_preserve('{ "a": 1, "b":2 }','{ "a": 3, "c":4 }');
+---------------------------------------------+--------------------------------------------------------------+
| json_merge_preserve('{"a": 1}', '{"b": 2}') | json_merge_preserve('{ "a": 1, "b":2 }','{ "a": 3, "c":4 }') |
+---------------------------------------------+--------------------------------------------------------------+
| {"a": 1, "b": 2}                            | {"a": [1, 3], "b": 2, "c": 4}                                |
+---------------------------------------------+--------------------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_MERGE(json_doc, json_doc[, json_doc] ...)**

与 JSON_MERGE_PRESERVE 作用一样，从 MySQL 8.0.3 开始不建议使用，后续会移除。



### 7.3 其它辅助函数

**JSON_QUOTE(string)**

生成有效的 JSON 字符串，主要是对一些特殊字符（如双引号）进行转义。

```sql
mysql> select json_quote('null'), json_quote('"null"'), json_quote('[1, 2, 3]');
+--------------------+----------------------+-------------------------+
| json_quote('null') | json_quote('"null"') | json_quote('[1, 2, 3]') |
+--------------------+----------------------+-------------------------+
| "null"             | "\"null\""           | "[1, 2, 3]"             |
+--------------------+----------------------+-------------------------+
1 row in set (0.00 sec)
```

除此之外，也可通过 CAST(value AS JSON) 进行类型转换。



**JSON_UNQUOTE(json_val)**

将 JSON 转义成字符串输出。

```sql
mysql> select c2->'$.ename',json_unquote(c2->'$.ename'),
    -> json_valid(c2->'$.ename'),json_valid(json_unquote(c2->'$.ename')) from t;
+---------------+-----------------------------+---------------------------+-----------------------------------------+
| c2->'$.ename' | json_unquote(c2->'$.ename') | json_valid(c2->'$.ename') | json_valid(json_unquote(c2->'$.ename')) |
+---------------+-----------------------------+---------------------------+-----------------------------------------+
| "jack"        | jack                        |                         1 |                                       0 |
| "mark"        | mark                        |                         1 |                                       0 |
+---------------+-----------------------------+---------------------------+-----------------------------------------+
2 rows in set (0.00 sec)
```

直观地看，没加 JSON_UNQUOTE 字符串会用双引号引起来，加了 JSON_UNQUOTE 就没有。但本质上，前者是 JSON 中的 STRING 类型，后者是 MySQL 中的字符类型，这一点可通过 JSON_VALID 来判断。



**JSON_OBJECTAGG(key, value)**

取表中的两列作为参数，其中，第一列是 key，第二列是 value，返回 JSON 对象。如，

```lua
mysql> select * from emp;
+--------+----------+--------+
| deptno | ename    | sal    |
+--------+----------+--------+
|     10 | emp_1001 | 100.00 |
|     10 | emp_1002 | 200.00 |
|     20 | emp_1003 | 300.00 |
|     20 | emp_1004 | 400.00 |
+--------+----------+--------+
4 rows in set (0.00 sec)

mysql> select json_objectagg(ename,sal) from emp;
+----------------------------------------------------------------------------------+
| json_objectagg(ename,sal)                                                        |
+----------------------------------------------------------------------------------+
| {"emp_1001": 100.00, "emp_1002": 200.00, "emp_1003": 300.00, "emp_1004": 400.00} |
+----------------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select deptno,json_objectagg(ename,sal) from emp group by deptno;
+--------+------------------------------------------+
| deptno | json_objectagg(ename,sal)                |
+--------+------------------------------------------+
|     10 | {"emp_1001": 100.00, "emp_1002": 200.00} |
|     20 | {"emp_1003": 300.00, "emp_1004": 400.00} |
+--------+------------------------------------------+
2 rows in set (0.00 sec)
```



**JSON_ARRAYAGG(col_or_expr)**

将列的值聚合成 JSON 数组，注意，JSON 数组中元素的顺序是随机的。

```sql
mysql> select json_arrayagg(ename) from emp;
+--------------------------------------------------+
| json_arrayagg(ename)                             |
+--------------------------------------------------+
| ["emp_1001", "emp_1002", "emp_1003", "emp_1004"] |
+--------------------------------------------------+
1 row in set (0.00 sec)

mysql> select deptno,json_arrayagg(ename) from emp group by deptno;
+--------+--------------------------+
| deptno | json_arrayagg(ename)     |
+--------+--------------------------+
|     10 | ["emp_1001", "emp_1002"] |
|     20 | ["emp_1003", "emp_1004"] |
+--------+--------------------------+
2 rows in set (0.00 sec)
```



**JSON_PRETTY(json_val)**

将 JSON 格式化输出。

```haskell
mysql> select json_pretty("[1,3,5]");
+------------------------+
| json_pretty("[1,3,5]") |
+------------------------+
| [
  1,
  3,
  5
]      |
+------------------------+
1 row in set (0.00 sec)

mysql> select json_pretty('{"a":"10","b":"15","x":"25"}');
+---------------------------------------------+
| json_pretty('{"a":"10","b":"15","x":"25"}') |
+---------------------------------------------+
| {
  "a": "10",
  "b": "15",
  "x": "25"
}   |
+---------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_STORAGE_FREE(json_val)**

MySQL 8.0 新增的，与 Partial Updates 有关，用于计算 JSON 文档在进行部分更新后的剩余空间。



**JSON_STORAGE_SIZE(json_val)**

MySQL 5.7.22 引入的，用于计算 JSON 文档的空间使用情况。



**JSON_DEPTH(json_doc)**

返回 JSON 文档的最大深度。对于空数组，空对象，标量值，其深度为 1。

```sql
mysql> select json_depth('{}'),json_depth('[10, 20]'),json_depth('[10, {"a": 20}]');
+------------------+------------------------+-------------------------------+
| json_depth('{}') | json_depth('[10, 20]') | json_depth('[10, {"a": 20}]') |
+------------------+------------------------+-------------------------------+
|                1 |                      2 |                             3 |
+------------------+------------------------+-------------------------------+
1 row in set (0.00 sec)
```



**JSON_LENGTH(json_doc[, path])**

返回 JSON 文档的长度，其计算规则如下：

1. 如果是标量值，其长度为 1。
2. 如果是数组，其长度为数组元素的个数。
3. 如果是对象，其长度为对象元素的个数。
4. 不包括嵌套数据和嵌套对象的长度。

```sql
mysql> select json_length('"abc"');
+----------------------+
| json_length('"abc"') |
+----------------------+
|                    1 |
+----------------------+
1 row in set (0.00 sec)

mysql> select json_length('[1, 2, {"a": 3}]');
+---------------------------------+
| json_length('[1, 2, {"a": 3}]') |
+---------------------------------+
|                               3 |
+---------------------------------+
1 row in set (0.00 sec)

mysql> select json_length('{"a": 1, "b": {"c": 30}}');
+-----------------------------------------+
| json_length('{"a": 1, "b": {"c": 30}}') |
+-----------------------------------------+
|                                       2 |
+-----------------------------------------+
1 row in set (0.00 sec)

mysql> select json_length('{"a": 1, "b": {"c": 30}}', '$.a');
+------------------------------------------------+
| json_length('{"a": 1, "b": {"c": 30}}', '$.a') |
+------------------------------------------------+
|                                              1 |
+------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_TYPE(json_val)**

返回 JSON 值的类型。

```sql
mysql> select json_type('123');
+------------------+
| json_type('123') |
+------------------+
| INTEGER          |
+------------------+
1 row in set (0.00 sec)

mysql> select json_type('"abc"');
+--------------------+
| json_type('"abc"') |
+--------------------+
| STRING             |
+--------------------+
1 row in set (0.00 sec)

mysql> select json_type(cast(now() as json));
+--------------------------------+
| json_type(cast(now() as json)) |
+--------------------------------+
| DATETIME                       |
+--------------------------------+
1 row in set (0.00 sec)

mysql> select json_type(json_extract('{"a": [10, true]}', '$.a'));
+-----------------------------------------------------+
| json_type(json_extract('{"a": [10, true]}', '$.a')) |
+-----------------------------------------------------+
| ARRAY                                               |
+-----------------------------------------------------+
1 row in set (0.00 sec)
```



**JSON_VALID(val)**

判断给定值是否是有效的 JSON 文档。

```sql
mysql> select json_valid('hello'), json_valid('"hello"');
+---------------------+-----------------------+
| json_valid('hello') | json_valid('"hello"') |
+---------------------+-----------------------+
|                   0 |                     1 |
+---------------------+-----------------------+
1 row in set (0.00 sec)
```



**JSON_TABLE(expr, path COLUMNS (column_list) [AS] alias)**

从 JSON 文档中提取数据并以表格的形式返回。

该函数的完整语法如下：

```less
JSON_TABLE(
    expr,
    path COLUMNS (column_list)
)   [AS] alias

column_list:
    column[, column][, ...]

column:
    name FOR ORDINALITY
    |  name type PATH string_path [on_empty] [on_error]
    |  name type EXISTS PATH string_path
    |  NESTED [PATH] path COLUMNS (column_list)

on_empty:
    {NULL | DEFAULT json_string | ERROR} ON EMPTY

on_error:
    {NULL | DEFAULT json_string | ERROR} ON ERROR
```

其中，

- expr：可以返回 JSON 文档的表达式。可以是一个标量（ JSON 文档 ），列名或者一个函数调用（ JSON_EXTRACT(t1.json_data,'$.post.comments') ）。

- path：JSON 的路径表达式，

- column：列的类型，支持以下四种类型：

- - name FOR ORDINALITY：序号。name 是列名。
  - name type PATH string_path [on_empty] [on_error]：提取指定路径（ string_path ）的元素。name 是列名，type 是 MySQL 中的数据类型。
  - name type EXISTS PATH string_path：指定路径（ string_path ）的元素是否存在。
  - NESTED [PATH] path COLUMNS (column_list)：将嵌套对象或数组与来自父对象或数组的 JSON 值扁平化为一行输出。

```sql
select *
 from
   json_table(
     '[{"x":2, "y":"8", "z":9, "b":[1,2,3]}, {"x":"3", "y":"7"}, {"x":"4", "y":6, "z":10}]',
     "$[*]" columns(
       id for ordinality,
       xval varchar(100) path "$.x",
       yval varchar(100) path "$.y",
       z_exist int exists path "$.z",
       nested path '$.b[*]' columns (b INT PATH '$')
     )
   ) as t;
+------+------+------+---------+------+
| id   | xval | yval | z_exist | b    |
+------+------+------+---------+------+
|    1 | 2    | 8    |       1 |    1 |
|    1 | 2    | 8    |       1 |    2 |
|    1 | 2    | 8    |       1 |    3 |
|    2 | 3    | 7    |       0 | NULL |
|    3 | 4    | 6    |       1 | NULL |
+------+------+------+---------+------+
5 rows in set (0.00 sec)
```



**JSON_SCHEMA_VALID(schema,document)**

判断 document （ JSON 文档 ）是否满足 schema （ JSON 对象）定义的规范要求。完整的规范要求可参考 [Draft 4 of the JSON Schema specification](https://json-schema.org/specification-links.html#draft-4) 。如果不满足，可通过 JSON_SCHEMA_VALIDATION_REPORT() 获取具体的原因。

以下面这个 schema 为例。

```swift
set @schema = '{
   "type": "object",
   "properties": {
     "latitude": {
       "type": "number",
       "minimum": -90,
       "maximum": 90
     },
     "longitude": {
       "type": "number",
       "minimum": -180,
       "maximum": 180
     }
   },
   "required": ["latitude", "longitude"]
}';
```

它的要求如下：

1. document 必须是 JSON 对象。
2. JSON 对象必需的两个属性是 latitude 和 longitude。
3. latitude 和 longitude 必须是数值类型，且两者的大小分别在 -90 ～ 90，-180 ～ 180 之间。

下面通过具体的 document 来测试一下。

```java
mysql> set @document = '{"latitude": 63.444697,"longitude": 10.445118}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_schema_valid(@schema, @document);
+---------------------------------------+
| json_schema_valid(@schema, @document) |
+---------------------------------------+
|                                     1 |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> set @document = '{"latitude": 63.444697}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_schema_valid(@schema, @document);
+---------------------------------------+
| json_schema_valid(@schema, @document) |
+---------------------------------------+
|                                     0 |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> select json_pretty(json_schema_validation_report(@schema, @document))\G
*************************** 1. row ***************************
json_pretty(json_schema_validation_report(@schema, @document)): {
  "valid": false,
  "reason": "The JSON document location '#' failed requirement 'required' at JSON Schema location '#'",
  "schema-location": "#",
  "document-location": "#",
  "schema-failed-keyword": "required"
}
1 row in set (0.00 sec)

mysql> set @document = '{"latitude": 91,"longitude": 0}';
Query OK, 0 rows affected (0.00 sec)

mysql> select json_schema_valid(@schema, @document);
+---------------------------------------+
| json_schema_valid(@schema, @document) |
+---------------------------------------+
|                                     0 |
+---------------------------------------+
1 row in set (0.00 sec)

mysql> select json_pretty(json_schema_validation_report(@schema, @document))\G
*************************** 1. row ***************************
json_pretty(json_schema_validation_report(@schema, @document)): {
  "valid": false,
  "reason": "The JSON document location '#/latitude' failed requirement 'maximum' at JSON Schema location '#/properties/latitude'",
  "schema-location": "#/properties/latitude",
  "document-location": "#/latitude",
  "schema-failed-keyword": "maximum"
}
1 row in set (0.00 sec)
```

#  

# 八、总结

如果要使用 JSON 类型，推荐使用 MySQL 8.0。相比于 MySQL 5.7，Partial update 带来的性能提升还是十分明显的。

Partial update 在存储引擎层是默认开启的，binlog 中是否开启取决于 binlog_row_value_options 。该参数默认为空，不会开启 Partial update，建议设置为 PARTIAL_JSON。

注意使用 Partial update 的前提条件。

当我们使用 JSON_CONTAINS、MEMBER OF、JSON_OVERLAPS 进行数组相关的操作时，可使用 MySQL 8.0.17 引入的多值索引来加快查询。

#  

# 九、参考资料

1. [JSON](https://zh.wikipedia.org/wiki/JSON)
2. [The JSON Data Type](https://dev.mysql.com/doc/refman/8.0/en/json.html)
3. [JSON Functions](https://dev.mysql.com/doc/refman/8.0/en/json-functions.html)
4. [Upgrading JSON data stored in TEXT columns](https://dev.mysql.com/blog-archive/upgrading-json-data-stored-in-text-columns/)
5. [Indexing JSON documents via Virtual Columns](https://dev.mysql.com/blog-archive/indexing-json-documents-via-virtual-columns/)
6. [Partial update of JSON values](https://dev.mysql.com/blog-archive/partial-update-of-json-values/)
7. [MySQL 8.0: InnoDB Introduces LOB Index For Faster Updates](https://dev.mysql.com/blog-archive/mysql-8-0-innodb-introduces-lob-index-for-faster-updates/)