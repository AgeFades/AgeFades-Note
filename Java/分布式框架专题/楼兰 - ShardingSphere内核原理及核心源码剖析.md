[TOC]

# 楼兰 - ShardingSphere内核原理及源码剖析

## 内核剖析

- ShardingSphere 各个产品的 **数据分片主要流程** 是完全一致的 

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/F8BC0B79A20842788809C1E0B589F402?ynotemdtimestamp=1631777836235)

### 解析引擎

- 解析过程分为：

  - 词法解析
    - 用于将 SQL 拆解为不可再分的原子符号，称为 **Token**
    - 再根据不同数据库方言所提供的字典，将其归类为：
      - 关键字
      - 表达式
      - 字面量
      - 操作符
  - 语法解析
    - 将 词法解析 的SQL结果 转换为 **抽象语法树(AST)**

- 举例：

```sql
# 该 SQL 会被解析成下面的树
SELECT id, name FROM t_user WHERE status = 'ACTIVE' AND age > 18
```

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/FB87BDC8E1934E3299BAEF7658E9E72B?ynotemdtimestamp=1631777836235)

- 为了便于理解，抽象树各颜色表示如下：
  - 绿色：关键字的Token
  - 红色：变量的Token
  - 灰色：表示需要进一步拆分
- 通过对 抽象语法树 的遍历，可以标记出有可能需要改写的位置（ 即将逻辑库(表)替换成真实库(表) ）
- SQL的一次解析过程是不可逆的，所有 token 按 sql 原本的顺序，依次进行解析，性能很高
- 在解析过程中，需要考虑各种数据库SQL方言的异同，提供不同的解析模板

#### SQL解析引擎

- **SQL解析是整个分库分表产品的核心**
  - 性能和兼容性是最重要的衡量指标
- ShardingSphere1.4.x 之前，采用的是性能较快的 Druid 作为 SQL解析器
  - 1.5.x 后，采用自研SQL解析器
  - 3.0.x后，使用 **ANLTR** 作为 SQL解析引擎
- SQL解析整体结构如下：

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/EB7FF7CA25394898A7CFC1FAE33E03C5?ynotemdtimestamp=1631777836235)

### 路由引擎

#### 简介

- 根据 **解析上下文匹配数据库和表的分片策略，生成路由路径**

#### 分片策略

- 主要路由方式
  - 单片路由（分片键的操作符是等号）
  - 多片路由（分片键的操作符是 IN）
  - 范围路由（分片键的操作符是 Between）
  - 广播路由（不携带分片键的SQL）

- 分片策略通常分为：
  - 数据库内置
    - 尾数取模
    - 哈希
    - 范围
    - 标签
    - 时间
  - 自定义配置
    - 灵活配置

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/E4C9EE9388B34517AC4E63329B9DA82B?ynotemdtimestamp=1631777836235)

- 全库表路由：
  - 对于不带分片键的 DQL、DML、DDL 语句
  - 会遍历所有库表，注意执行
  - 例如：select * from user;
- 全库路由：
  - 对数据库的操作都会遍历所有真实库
  - 例如：set autocommit = 0;
- 全实例路由：
  - 对于 DCL 语句，每个数据库实例只执行一次
  - 例如：create user customer@127.0.0.1 identified by '123';
- 单播路由：
  - 仅需从任意库中获取数据即可
  - 例如：describe course;
- 阻断路由：
  - 屏蔽SQL对数据库的操作（不会执行）
  - 例如：use testdb;

### 改写引擎

- 开发只需要面向 逻辑库 和 逻辑表 来写 SQL
  - 最终由 ShardingSphere 的改写引擎将 SQL 改写成
  - 在真实库表中，正确执行的 SQL
- SQL改写分为：
  - 正确性改写
  - 优化改写

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/C93215250BF14C03A7B3F340D8B987CC?ynotemdtimestamp=1631777836235)

### 执行引擎

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/8081284FC58B4970B40C8A05C1EFF158?ynotemdtimestamp=1631777836235)

- ShardingSphere 并不是简单的将改写完的 SQL 提交到数据库执行
- 执行引擎的目的：
  - 自动化的平衡资源控制和执行效率

#### 连接模式

- 内存限制模式：MEMORY_STRICTLY
  - 只关注 **一个数据库连接的处理数量**，通常 **一张真实表一个数据库连接**
  - 不限制连接数，即会建立多个数据连接
    1. 并发控制每个连接 只去读取一个数据分片的数据，这样可以最快将所有需要的数据读出来
    2. 在后面归并阶段，会选择以每一条数据为单位进行归并（流式归并）
       1. 归并完一批数据，即可释放内存
       2. 这样**提高数据归并效率**，**有效防止 内存溢出 或 GC频繁**
    3. 吞吐量大，适合 OLAP 场景
- 连接限制模式：CONNECTION_STRICTLY
  - 只关注 **数据库连接的数量**，较大的查询会进行串行操作
  - 限制连接数，即至少有一个数据库连接、会去读取多个数据分片的数据
    1. 串行读所有分片数据、全部读到内存、统一归并（内存归并）
    2. 归并效率高，例如 MAX归并，内存中一次就能拿到最大值
       1. 而 流式归并 需要一条条比较
    3. 适合 OLTP 场景
  
- 这两个模式涉及到一个参数

  - ```yaml
    spring:
    	shardingsphere:
    		props:
    			max:
    				connections:
    					size:
    						per:
    							# 默认值为1
    							# 参见源码 ConfigurationPropertyKey
    							query: 50
    ```

  - ShardingSphere 会根据路由到某一个数据源的路由结果
  
    - 计算出 **所有需在数据库上执行的SQL数量**
    - 用这个 数量 / 用户的配置项 = 每个数据库连接需执行的SQL数量
    - 数量 > 1：连接限制模式
    - 数量 <= 1：内存限制模式

### 归并引擎

- 结果归并：
  - 将从各个数据节点获取的 多数据结果集，
  - 组成一个结果集、并正确的返回至请求客户端
- 流式归并：
  - 一条一条数据的方式进行归并
- 内存归并：
  - 将所有结果集都查询到内存中，进行统一归并

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/59D29420DF0F4C79B24A9C888645AE7E?ynotemdtimestamp=1631777836235)

- 举例：
  - AVG归并就无法直接进行分片归并，
  - 需要转化成 COUNT$SUM 的累加归并，
  - 然后再计算平均值
- 排序归并的流程图如下：

![](https://note.youdao.com/yws/public/resource/69180c50e1a1d23df85f80ef15517d4b/1426572B95F84B899A38A7DB53251F12?ynotemdtimestamp=1631777836235)

## ShardingSphere中的SPI

[官方文档 - ShardingSphere 支持的SPI扩展点](https://shardingsphere.apache.org/document/current/cn/dev-manual/)

## ShardingJDBC源码流程图

[ShardingJDBC源码流程图](https://www.processon.com/view/link/61444513e0b34d66cf86729c)

