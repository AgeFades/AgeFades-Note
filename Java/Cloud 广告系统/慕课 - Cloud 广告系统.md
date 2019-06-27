# 慕课 - Cloud 广告系统

## 课程导学

### 	什么是广告系统

​		广告计费系统

​		广告投放系统

​		广告检索系统

​		报表系统

### 	重点

​		怎样定义投放系统的数据结构？

​		怎样高效的处理广告请求？

### 	知识点

​		SpringCloud

​		广告建搜

​		微服务架构

​		解析 MySQL Binlog

## 广告系统概览

### 	实现功能

![UTOOLS1561608499918.png](https://i.loli.net/2019/06/27/5d144135268ca57994.png)

### 	包含哪些子系统

![UTOOLS1561612498032.png](https://i.loli.net/2019/06/27/5d1450d33825d15150.png)

### 	广告系统架构

![image-20190627131558898](/Users/admin/Library/Application Support/typora-user-images/image-20190627131558898.png)

## 项目搭建

​	此处略去 maven 项目搭建、Eureka 搭建....

​	mvn clean package -Dmaven.test.skip=true -U

### 	微服务架构的两种方式

![UTOOLS1561613347762.png](https://i.loli.net/2019/06/27/5d14542501f2b84684.png)

![UTOOLS1561613392236.png](https://i.loli.net/2019/06/27/5d1454509612a83963.png)	

### 	Zuul 的声明周期

![UTOOLS1561613523015.png](https://i.loli.net/2019/06/27/5d1454d3b2dfa65970.png)

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

```java
@EnableZuulProxy // 启动类上注解
```

### 	自定义网关过滤器的开发

```java
/**
 * 网关过滤器
 */
@Slf4j
@Component
public class PreRequestFilter extends ZuulFilter {
	
    /**
	 * 过滤器类型
	 * 具体参照 FilterConstants 常量类型
	 */
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    /**
     * 过滤器优先级，越小优先级越高
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 是否需要过滤器，自定义判断逻辑
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 改 Filter 需要执行的具体操作
     */
    @Override
    public Object run() throws ZuulException {

        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set("startTime", System.currentTimeMillis());

        return null;
    }
}
```

## 通用模块

### 	通用代码、配置

### 	统一响应机制

### 	统一异常处理

​	   此处略。

## 系统开发

### 	IOC

![UTOOLS1561614637024.png](https://i.loli.net/2019/06/27/5d14592f2109b88625.png)

### 	MVC

![UTOOLS1561614697546.png](https://i.loli.net/2019/06/27/5d14596a9414648635.png)

### 	SpringBoot 常用功能

​		区分环境

```yaml
spring:
	profiles:
 		active: prod # or dev | test 然后编写 application-prod.yml ..
```

### 	广告投放子系统

```xml
		<!-- 引入 Feign, 可以以声明的方式调用微服务 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <!-- 引入服务容错 Hystrix 的依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
        <!-- 引入服务消费者 Ribbon 的依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
```

```yaml
spring:
	jpa:
		show-sql: true # 展示 SQL
		hibernate:
			ddl-auto: update # 启动时更新表结构
		properties:
			hibernate.format_sql: true # 格式化 SQL
		open-in-view: false # 关闭懒加载
	
```

```java
@EnableFeignClients // 启用 Feign
@EnableCircuitBreaker // 断路器
```

### 	投放系统在网关中的配置

```yaml
zuul:
	prefix: /xx # 走网关的前缀 例 : www.xxx.com/api，此处即为 api
	routes: 
		xxx: # 需要路由的服务
			path: /xxx/** # 路由服务的路径
			serviceId: xxx # 此处为微服务的 spring.application.name 	
			strip-prefix: false # 不跳过前缀 例 : 127.0.0.1:8080/api/user/create
			
```

### 	MySQL 慢查询

#### 		介绍

​			慢查询日志记录 MySQL 中响应时间超过阙值（long_query_time）的SQL语句。

​			默认 : 10s ，未启动。

​			企业开发需要打开，有一定性能影响。

​			支持记录到表，也支持记录到库。

#### 		相关变量

```sql
show variables like 'slow_query_log';  -- 是否开启慢查询日志
show variables like 'slow_query_log_file'; -- 慢查询日志存储路径(MySQL 不同版本不同路径)
show variables like 'long_query_time'; -- 慢查询阙值
show variables like 'log_queries_not_using_indexes'; -- 未使用索引的查询是否也被记录到慢查询日志中
show variables like 'log_output'; -- 慢查询日志存储方式	默认文件
show global status like 'Slow_queries'; -- 查看有多少条慢查询记录
```

#### 		修改变量

​			例 : set global long_query_time = 0.2; 

​			更常见的做法是 vim my.cnf

#### 		慢查询日志分析工具

##### 			mysqldumpslow

```mysql
# 使用语法
mysqldumpslow [options] [log_file ...]

# options
-g pattern:只显示与模式匹配的语句，大小写不敏感。
-r:反转排序顺序。
-s sort_type:如何排序输出，可选的 sort_type 如下
	t:按查询总时间排序。
	l:按查询总锁定时间排序。
	r:按总发送行排序。
	c:按计数排序。
	at:按查询时间或平均查询时间排序。
    al:按平均锁定时间排序。
    ar:按平均行发送排序。
默认情况下，mysqldumpslow 按平均查询时间(相当于-s at)排序。
-t N:是 top n 的意思，即返回前面多少条的数据。
-v:详细模式。

# 使用示例
# 显示 2 条结果，且按照查询总时间排序，且过滤 group by 语句
mysqldumpslow -t 2 -s t -g "group by" slow_query_log_file

# 按照时间排序的前 10 条里面含有 左连接 的查询语句
mysqldumpslow -s t -t 10 -g "left join" slow_query_log_file

# 返回记录集最多的 10 个SQL
mysqldumpslow -s r -t 10 slow_query_log_file

# 可以结合 more 一起使用，避免一次性显示过多 SQL
mysqldumpslow -s r -t 20 slow_query_log_file | more

# 访问次数最多的 10 个SQL
mysqldumpslow -s c -t 10 slow_query_log_file

# 结果信息
Count # 这种类型的语句执行了几次
Time # 这种类型的语句执行的最大时间
Lock # 执行时等待缩的时间
Rows # 单次返回的结果数

# 结果示例
Count: 2 Time=3.21s (7s) Lock=0.00s (0s) Rows=1.0 (2), root[root]@localhost     
执行了 2 次，最大时间是 3.21s，总共花费时间 7s，等待锁的时间是 0s，单次返回的结果数是 1 条记录，2 次总共返回 2 条记录。

```

##### 			EXPLAIN

```mysql
# 使用方法
explain select * from xx where xx like '%x';

# 结果信息
id # 查询的唯一标识符
select_type # 查询的类型
table # 查询的表
partitions # 匹配的分区
type # join 类型
possible_keys # 查询中可能用到的索引
key # 查询中使用到的索引
key_len # 查询优化器使用了的索引字节数
ref # 哪个字段或常量与 key 一起被使用
rows # 当前的查询一共扫描了多少行
filtered # 查询条件过滤的数据百分比
Extra # 额外信息
```

### 	MySQL 索引

#### 		索引类型

```mysql
# 普通索引，支持单列和多列
# 创建索引方式
CREATE INDEX xx(索引名) ON xx(表名) (xx,xx....(列名));
ALTER TABLE xx(表名) ADD INDEX xx(索引名) (xx,xx...(列名));
CREATE TABLE xx(表名) ([...],INDEX 索引名 (xx,xx...));

# 唯一索引
CREATE UNIQUE INDEX ... # 三种皆同上，在 INDEX 前加 UNIQUE

# 主键索引
PRIMARY KEY .. # 两种同上，不支持 CREATE INDEX 方式

# 索引操作
# 删除索引
DROP INDEX xx(索引名) ON xx(表名)

# 查看索引
show index from xx(表名)
```

#### 		数据结构	

​			B+ Tree 

​				非叶子节点不存储数据，只存储索引

​				叶子节点包含全部的关键字信息，且叶子节点按照关键字顺序相互连接

![UTOOLS1561619643202.png](https://i.loli.net/2019/06/27/5d146cbe4aa4664944.png)

### 	MySQL 事物隔离级别

​		略

## 微服务调用

### 	广告检索系统子模块

```yaml
feign:
	hystrix:
		enabled: true # 开启 Fein + hystrix
managment:
	endpoints:
		web:
			exposure:
				include: "*" # 暴露服务全部信息给监控
		
```

### 	基于 Ribbon 实现微服务调用

```java
/**
 * 在启动类添加 Bean
 */
@Bean
@LoadBalanced // ribbon 实现负载均衡
public RestTemplate restTemplate(){
    return new RestTemplate();
}

/**
 * 在 Controller 中通过 restTemplate调用服务
 */
restTemplate.xx(请求方式)(
	xx(url), // url 可以通过注册在服务中心的服务名代替 ip + port
    xx(resqust信息),
    xx(序列化信息)
)
```

### 	基于 Feign 实现微服务调用

```java
/**
 * 定义 Feign 接口
 */
@FeignClient(value = "xx",fallback = xxClientHystrix.class) // 指向 Feign 需要调用的服务名称
public interface xxClient {
    
    // 需要调用服务的 Controller 方法
    // 例如
    @PostMapping(value = "/findById/{id}")
    ResultVo findById(@PathVariable Long id);
    
}

/**
 * Controller 中 @Autowire 引入 Feign 接口直接调用
 */
@PostMapping(value = "/findById/{id}")
ResultVo findById(@PathVariable Long id){
    return xxClient.findById(id);
}

/**
 * Hystrix
 */
@Component
public class xxClientHystrix implements xxClient{
    @Oveeride
    ResultVo findById(@PathVariable Long id){
        return "error";
    }
}
```

## 广告数据索引的设计

### 	正向索引

​		通过 唯一键/主键 生成与对象的映射关系

![UTOOLS1561621343534.png](https://i.loli.net/2019/06/27/5d147360d420913096.png)

### 	倒排索引（反向索引）

​		存储在全文搜索下某个单词在一个文档或一组文档中 存储位置 的映射

​		是 文档检索系统 中最常用的数据结果

![UTOOLS1561621474331.png](https://i.loli.net/2019/06/27/5d1473e33ce3e51447.png)

### ​	全量索引 

​		检索系统在启动时一次性读取当前数据库中的所有数据，建立索引。

### 	增量索引

​		系统运行过程中，监控数据库变化，即增量，实时加载更新，构建索引。	

### 	系统索引设计

​		这教程教你本地写索引文件.. 个人感觉没有必要，略

## 结束

​	后续主要是对 Binlog 日志记录后提取本地文件

​	可作为索引、日志分析.. 等等微服务

​	Kafka 的介绍

​	Hystrix Dashboad : 一个熔断器的监控

​	整个系统的可用性测试 Mork 数据

​	以上感觉没有太多必要进行记录，略。		