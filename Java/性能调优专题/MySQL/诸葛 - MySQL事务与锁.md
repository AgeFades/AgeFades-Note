[TOC]

# 诸葛 - MySQL 事务与锁

## 概述

```shell
# MySQL 中一般会并发执行多个事务，
	# 多个事务可能会 并发的对相同数据 进行 CRUD 操作，
	
	# 可能导致 脏写、脏读、不可重复读、幻读 等问题。
	
	# 这些问题本质上来说，都算是 数据库的多事务并发问题。
	
# 为了解决 多事务并发问题，
	# 数据库设计了:
		# 事务隔离机制
		
		# 锁机制
		
		# MVCC 多版本并发控制隔离机制
		
	# 用一整套机制来 解决多事务并发问题。
```

## 事务

```sql
-- 一组 SQL 的原子操作，要么都成功，要么都失败
BEGIN;
sql1
sql2
sql3
COMMIT;
```

### ACID

```shell
# 原子性(Atomicity)
	# 事务是一个 原子操作单元，其对数据的修改，要么 全部执行成功，要么 全部执行失败。
	
	# 比如: 下单 -> 支付 -> 减少库存
		# 这三个操作 在一个事务中，要么全部执行成功，要么全部执行失败

# 一致性(Consistent)
	# 在 事务开始和完成 时，数据都必须保持一致。
	
	# 比如: 创建订单 -> 支付扣款 -> 库存数量修改
		# 这三个对数据的操作是必须一致的，
		
		# 可以理解为 原子性 的 数据层面。

# 隔离性(Isolation)
	# 事务处理过程中的中间状态 对外部是不可见的。

# 持久性(Durable)
	# 事务完成之后，对 数据的修改 是永久性的。
```

### 并发事务带来的问题

#### 脏写

```shell
# A、B 两个事务同时开启，更新 t1.id 字段
	# A 提交 t1.id = 2
	
	# B 提交 t1.id = 3
	
	# 最终 t1.id = 3，A 更新丢失（Lost Update）
	
	# 即: 最后的更新覆盖了之前其他事务的更新。
```

#### 脏读

```shell
# A、B 两个事务同时开启，使用 t1.id 字段
	# A 更新 t1.id = 2，未提交
	
	# B 读取 t1.id = 2 并用于使用
	
	# A 更新 t1.id = 3，提交事务
	
	# B 读取到的 t1.id = 2 实际是脏数据
	
	# 即: 事务A 读取到了 事务B 已经修改
```

#### 不可重读

```shell
# A、B 两个事务同时开启，使用 t1.id 字段
	# A 更新 t1.id = 2，未提交 
	
	# B 读取 t1.id = 2
	
	# A 更新 t1.id = 3
	
	# B 读取 t1.id = 3 ？？ 到底是 2 还是 3？
	
	# 即: 事务内部中相同查询语句在不同时间读取到的结果不一致，不符合隔离性
```

#### 幻读

```shell
# A、B 两个事务同时开启，使用 t1 表
	# A 新增数据，提交事务
	
	# B 未提交，读取到了 A 新增的数据
	
	# 即: 事务 A 读取到了 事务 B 提交的新增数据，不符合隔离性
```

### 事务隔离级别

```shell
# 以上问题都是 数据库读一致性 问题，必须由 数据库 提供一定的事务隔离机制来解决。
```

| 隔离级别 | 脏读   | 不可重复读 | 幻读   |
| -------- | ------ | ---------- | ------ |
| 读未提交 | 可能   | 可能       | 可能   |
| 读已提交 | 不可能 | 可能       | 可能   |
| 可重复读 | 不可能 | 不可能     | 可能   |
| 可串行化 | 不可能 | 不可能     | 不可能 |

```shell
# 数据库的 事务隔离 越严格，并发副作用越小，但付出的代价也就越大，
	# 因为 事务隔离 实质上是就是使 事务 在一定程度上 "串行化" 进行，
	
	# 这显然与 "并发" 是矛盾的。
	
# 不同的应用对 读一致性 和 事务隔离程度 的要求也是不同的，
	# 比如许多应用对 "不可重复读" 和 "幻读" 并不敏感，
	
	# 可能更关心数据并发访问的问题。
```

```sql
-- 查看当前数据库的事务隔离级别
select @@transaction_isolation;

-- MySQL 默认的事务隔离级别是 可重复读
```

## 锁详解

```shell
# 锁 是计算机 协调多个进程或线程并发访问某一资源的机制。

# 在 数据库 中，除了传统的计算资源（如: CPU、RAM、I/O 等）的争用外，
	# 数据 也是一种 需要用户共享 的资源，
	
	# 锁 也用来保证 数据并发访问一致性、有效性，
	
	# 锁 也是影响 数据库并发访问性能 的一个重要因素。
```

### 锁分类

```shell
# 性能上分为: 乐观锁(版本对比实现) | 悲观锁

# 粒度上分为: 表锁 | 行锁

# 操作上分为: 读锁(悲观锁) | 写锁(悲观锁)

	# 读锁(共享锁): 
		# 针对 同一份数据，多个 读操作 可以同时进行而不会 互相影响
		
	# 写锁(独占锁):
		# 当前 写操作 没有完成前，阻断其他线程操作	
```

#### 表锁

```shell
# 每次操作锁住 整张表，
	# 开销小，加锁快，
	
	# 不会出现死锁，
	
	# 锁定力度大，
	
	# 发生 锁冲突 的概率最高，并发度最低，
	
	# 一般用在 整表数据迁移 的场景
```

##### 基本操作

```sql
-- 手动添加表锁
lock table [表名称1] read(write), [表名称2] read(write);

-- 查看表上加过的锁
show open tables;

-- 删除表锁
unlock tables;
```

##### 案例

```sql
-- 读锁
-- 所有会话均可读数据
-- 当前会话 对 数据进行操作 会报错，其他会话 对 数据进行操作会阻塞等待
lock table customer read;
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1600677967074.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1600677983569.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1600678196654.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1600678240296.png)

```sql
-- 写锁
-- 这里就懒得截图了
-- 当前会话 对该表的 crud 都没有问题，
-- 其他会话 对该表的 crud 都被阻塞
lock table customer write;
```

##### 结论

```shell
# 对 MyISAM 引擎来说，只支持 表级锁，
	# 可想而知，对于当前 互联网公司 性能至上 来说，基本没有上场的机会。
```

#### 行锁

```shell
# 每次操作锁住 一行数据，
	# 开销大，加锁慢（要先找到表的数据行，再去加锁，这个过程是耗费时间的）
	
	# 会出现死锁（id=1 等待 id=2 行解锁，id=2 等待 id=1 行解锁）
	
	# 锁定力度最小，发生 锁冲突 的概率最低，
	
	# 并发度最高。
```

##### InnoDB 和 MyISAM 最大不同

```shell
# InnoDB 支持事务

# InnoDB 支持行级锁
```

##### 演示结论

```shell
# 这里就懒得做演示了

# 一个会话 开启事务更新 不提交，
	# 另一个会话 更新同一条记录 会阻塞，
	
	# 更新不同记录不会阻塞。
```

#### 总结

```shell
# MyISAM 在执行 select 前，自动给所有 涉及表 加读锁
	# 执行 update、insert、delete 前，自动给 涉及表 加写锁。
	
# InnoDB 在执行 select 前（非串行化隔离级别），不会加锁
	# 执行 update、insert、delete 操作会加行锁。
```

## 行锁与事务隔离级别案例分析

```shell
# 懒得截图了，按下面 sql 顺序执行就行，0表示初始工作(不分先后)
```

### 演示SQL

```sql
create table `account` (
	`id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `balance` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `account` (`name`, `balance`)
VALUES
('张三', 300),
('李四', 400),
('王五', 500);
```

### 读未提交

```sql
-- Session1

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为读未提交
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 0. 开启事务
START TRANSACTION;

-- 1. 第一次读数据
select * from account;

-- 3. 第二次读数据
select * from account;
```

```sql
-- Session2

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为读未提交
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 0. 开启事务
START TRANSACTION;

-- 2. 事务内修改数据
update account set balance = balance - 50 where id = 1;
```

```shell
# 结论:

# Session1 读到了 Session2 事务未提交的修改数据，
	# 一旦 Session2 发生回滚或者继续变更数据，
	
	# Session1 读到的就是脏数据（脏读）
	
	# 想解决 脏读 就用 读已提交 
```

### 读已提交

```sql
-- Session1

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为读已提交
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 0. 开启事务
BEGIN;

-- 1. 第一次读数据
select * from account;

-- 4. 第二次读数据
select * from account;
```

```sql
-- Session2

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为读已提交
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 0. 开启事务
BEGIN;

-- 2. 事务内修改数据
update account set balance = balance - 50 where id = 1;

-- 3. 提交事务
COMMIT;
```

```shell
# 结论:
	# 读已提交 解决了 脏读 的问题
	
	# 没有解决 可重复读 的问题
		# 同一个事务内只执行 select 操作，不同时间段得到的结果不一致
```

### 可重复读

```sql
-- Session1

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为可重复读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 0. 开启事务
BEGIN;

-- 1. 第一次读数据
select * from account;

-- 4. 第二次读数据
select * from account;

-- 5. 事务内修改数据
update account set balance = balance - 50 where id = 1;

-- 6. 第三次读数据
select * from account;

-- 10. 事务内修改数据
update account set balance = balance + 50 where name = '赵六';

-- 11. 第四次读数据
select * from account;

-- 12. 提交事务
COMMIT;
```

```sql
-- Session2

-- 0. 查询当前事务隔离级别
select @@transaction_isolation;

-- 0. 设置当前会话事务隔离级别为读未提交
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 0. 开启事务
BEGIN;

-- 2. 事务内修改数据
update account set balance = balance - 50 where id = 1;

-- 3. 提交事务
COMMIT;

-- 7. 开启事务
BEGIN;

-- 8. 插入数据
INSERT account (`balance`, `name`) VALUES (600, '赵六'); 

-- 9. 提交数据
COMMIT;
```

```shell
# 结论:
	# 可重复读 解决了 脏读 + 不可重复读 的问题。
	
# 假设 Session2 三步执行完毕，balance 的值为 200，
	# 而此时 Session1 第四步读取到的 balance 值仍为 250，
	
	# 当 Session1 第五步执行完毕，执行第六步，读取到的 balance 值为 150
	
	# 这是由于 可重复读 使用了 MVCC（多版本并发控制机制），
		# select 操作不会更新版本号，是快照读（历史版本）
		
		# insert、update、delete 会更新版本号，是实时读（当前版本）
		
# 可重复读仍然可以存在 幻读 的问题
	# Session2 第九步执行完毕，
	
	# Session1 虽然直接 select 不到 Session2 新增的数据
		# 但是可以通过 update 的形式读出 Session2 新增数据
		
# 可重复度 已经基本足够满足我们的开发需求了，也是 默认的隔离级别
```

### 串行化

```shell
# 设置当前会话事务隔离级别为 串行化
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;

# 串行化就是，当一个事务进行 update、insert、delete 操作时，
	# 直接加行锁，
	
	# 管你其他事务是干啥的，反正都必须等当前事务提交之后才能操作。
	
	# 这种性能太差，基本上就不会用。
```

### 间隙锁(Gap Lock)

```shell
# 间隙锁 在 可重复度隔离级别 下才会生效。

# MySQL 事务隔离级别默认是 可重复度，
	# 间隙锁 在某些情况下可以解决 幻读 问题
```

```sql
-- Session1 操作区间数据
update account set name = '嘻嘻' where id > 1 and id < 10;

-- 其他会话则 不能 在该范围内所包含的 所有行记录 以及 中间的间隙 里 插入或修改任何数据
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602224600613.png)

```sql
-- 如上图，间隙就有 id 为 (3, 10) (10, 20) (20, 正无穷) 这三个区间。

-- 执行如下 SQL
update account set name = 'zhangsan' where id > 8 and id < 18;

-- 不提交事务，在另一个 Session 中插入 id = 7 或者 id = 19，发现都无法成功
-- 这就是 MySQL 在 可重复度隔离级别下 使用 间隙锁 解决部分 幻读 问题的证明。
-- 因为 id > 8 and id < 18，间隙锁的实际作用范围是 (3, 20]
```

### 临键锁(Next-key Locks)

```shell
# 临键锁是 行锁与间隙锁 的组合，

# 比如 id 为 (1,10] 的整个区间可以叫做临键锁。
```

### 无索引行锁会升级为表锁

```shell
# InnoDB 的 行锁 是 针对索引 加的锁，而非针对记录加的锁，
	# 如果对 非索引文件 字段更新，行锁就会变成表锁，
	
	# 并且 行锁针对索引 不能失效，否则也会从 行锁 升级为 表锁。
```

### 行锁操作

```sql
-- lock in share mode(读锁)

-- for update(写锁)

-- 示例sql，其他 session 只能读，其他操作会被阻塞
select * from account where id = 2 for update;
```

### 行锁分析

```sql
-- 检查 InnoDB_rol_lock 状态变量，可以分析系统上的行锁争夺情况
show status like 'innodb_row_lock%';
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1601195983877.png)

```shell
Innodb_row_lock_current_waits # 当前正在等待锁定的数量

Innodb_row_lock_time # 从 系统启动到现在 锁定总时间 长度（重要）

Innodb_row_lock_time_avg # 每次等待锁的平均时间（重要）

Innodb_row_lock_time_max # 从 系统启动到现在 等待最长的一次时间

Innodb_row_lock_waits # 从 系统启动到现在 总共等待的次数（重要）

# 当 等待次数 很高，并且 平均等待时长 也不低的时候，
	# 就需要分析系统中为什么会有这么多等待，
	
	# 根据分析结果着手制定优化计划。
```

### information_schema 系统库 锁相关数据表

```sql
-- 查看事务
select * from information_schema.INNODB_TRX;

-- 查看锁5.7
select * from information_schema.INNODB_LOCKS;

-- 查看锁 8
select * from `performance_schema`.data_locks;

-- 查看锁等待5.7
select * from information_schema.INNODB_LOCK_WAITS;

-- 查看锁等待 8 
select * from `performance_schema`.data_lock_waits;

-- 释放锁 trx_mysql_thread_id 来自 information_schema.INNODB_TRX
kill trx_mysql_thread_id

-- 查看锁等待详细信息
show engine innodb status \G;
```

### 死锁

```sql
-- Session1

-- 设置隔离级别 可重复读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 1. 查 id=1 并加行写锁
select * from account where id = 1 for update;

-- 3. 查 id=2 并加行写锁(阻塞)
select * from account where id = 2 for update;
```

```sql
-- Session2

-- 设置隔离级别 可重复读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 2. 查 id=2 并加行写锁
select * from account where id = 2 for update;

-- 4. 查 id=1 并加行写锁(阻塞)
select * from account where id = 1 for update;
```

```sql
-- 查看近期死锁日志信息
show engine innodb status \G;

-- 大多数情况，MySQL 可以自动检测 死锁 并回滚 产生死锁 的那个事务，但并不是所有场景。
```

### 锁优化建议

```shell
# 还是要按照实际业务来
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1601198602433.png)

### 结论

```shell
# InnoDB 实现了行锁，
	# 实现方面会比表锁更消耗性能，
	
	# 但在整体并发处理能力上，要远远优于 MyISAM 表锁。
	
# InnoDB 行锁也可能升级为表锁。
```

