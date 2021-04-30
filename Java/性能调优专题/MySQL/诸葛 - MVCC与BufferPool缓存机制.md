[TOC]

# 诸葛 - MVCC与BufferPool缓存机制

## MVCC 多版本并发控制机制

```shell
# MySQL 在 读已提交 和 可重复读 隔离级别下，实现了 MVCC 机制。

# 在 读已提交 和 可重复读 下，
	# 事务的 隔离性 是靠 MVCC机制 保证的，
	
	# 对 一行数据 的 读和写 两个操作，
	
	# 默认是不会通过 加锁互斥 来保持隔离性。
	
# 在 串行化 下，
	# 为了保证较高的隔离性，
	
	# 是通过将 所有操作 加锁互斥 来实现的。
```

### undo 日志版本链与 read view 机制详解

```shell
# undo 日志版本链，是指 一行数据被多个事务依次修改，
	# 在 每个事务修改完 后，
	
	# MySQL 会保留修改前的数据 undo 回滚日志，
	
	# 并且用两个 隐藏字段 trx_id 和 roll_pointer，
	
	# 把这些 undo 日志串联起来形成一个历史记录版本链
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1601273078778.png)

```shell
# 在 可重复读隔离级别，
	# 当 事务开启，执行任何查询SQL时，会生成 当前事务的一致性视图(read-view)，
	
	# 该视图在 事务结束 之前，都不会发生变化，
		# 如果是 读已提交，则 每次执行查询SQL，都会重新生成。
		
	# 该视图由 执行查询 时，所有未提交事务id数组(数组里最小的为 min_id)，
		# 和 已创建的最大事务id(max_id) 组成。
	
	# 事务里的任何 sql查询结果 需要从 对应版本链里的最新数据开始，
		# 逐条跟 read-view 做比对从而得到最终的快照结果。
```

#### 版本链对比规则

```shell
# 1. 如果 row 的 trx_id 落在绿色部分(trx_id < min_id),
	# 表示这个版本是由 已提交的事务 生成的，数据是可见的。
	
# 2. 如果 row 的 trx_id 落在红色部分(trx_id > max_id),
	# 表示这个版本是由 将来启动的事务生成的，是不可见的。
		# 如果 row 的 trx_id 就是当前自己的事务，是可见的。
		
# 3. 如果 row 的 trx_id 落在黄色部分(min_id <= trx_id <= max_id),
	# 3.1. 若 row 的 trx_id 在视图数组中，表示这个版本是由还没提交的事务生成的，不可见，
		# 如果 row 的 trx_id 就是当前自己的事务，是可见的。
	
	# 3.2. 若 row 的 trx_id 不在视图数组中，表示这个版本是已经提交的了事务生成的，可见。
	
# 对于删除的情况，可以认为是 update 的特殊情况，
	# 会将 版本链 上最新的数据复制一份，然后将 trx_id 修改成 删除操作的 trx_id，
	
	# 同时在 该条记录的头信息(record header) 里的 deleted_flag 标记位写上 true，
	
	# 意味着记录已被删除，不返回数据。
```

#### 注意

```shell
# begin 或者 start transaction 命令，并不是一个事务的起点，
	# 在执行到它们之后的 第一个修改操作 InnoDB表 语句，
	
	# 事务才真正启动，才会向 MySQL 申请事务id，
	
	# MySQL 内部是 严格按照事务的启动顺序 来分配事务id的。
```

#### 总结

```shell
# MVCC 机制的实现就是通过 read-view机制 与 undo版本链比对机制，
	# 使得 不同的事务 会根据 数据版本链对比规则 读取同一条数据在版本链上的不同版本数据。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602322734499.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602322754704.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602322765357.png)

## InnoDB 引擎SQL执行的BufferPool缓存机制

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1601277338501.png)

```shell
# BufferPool 内部也有一套 LRU 算法，
	# 所有 缓存 都会设计一套 数据淘汰策略
```

### 为什么不直接更新磁盘

```shell
# 如果来一个请求，就直接对磁盘文件进行随机读写，然后更新磁盘文件里的数据，性能相当差。

# MySQL 利用这套机制，可以保证每个更新请求都是 更新内存BufferPool，
	# 然后 顺序写日志文件，同时还能保证各种异常情况下的数据一致性。
	
	# 这也是 MySQL 在较高配置的机器上，每秒可以抗下几千QPS的原因。
	
# 主要差距就是: 顺序写日志 和 随机写磁盘 的性能巨大差异。
	# 为什么 db 数据不能顺序写？
		# 因为 db 数据会涉及到 删除，删除之后可能再插入其他数据，就不是 连续性 的数据。
		
		# 而 redo 日志是不会 删除 中间某条数据的。
```

