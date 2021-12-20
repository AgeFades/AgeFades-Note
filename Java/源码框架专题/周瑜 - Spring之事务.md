[TOC]

# 周瑜 - Spring之事务

## @EnabelTransactionManagement工作原理

- 开启Spring事务，本质上就是增加了一个 Advisor(切面)
- @EnabelTransactionManagement 功能：向Spring容器中添加了两个bean
  - AutoProxyRegistrar
  - ProxyTransactionManagementConfiguration

### AutoProxyRegistrar

- 主要作用：向Spring容器中注册了一个 `InfrastructureAdvisorAutoProxyCreator` 的 bean

#### InfrastructureAdvisorAutoProxyCreator

- 继承自 `AbstractAdvisorAutoProxyCreator`，也就是一个 BeanPostProcessor
- 主要作用：开启自动代理
  - 会在初始化后步骤，寻找 Advisor 类型的 bean
  - 并判断当前某个 bean 是否有匹配的 Advisor，是否需要利用动态代理 生产 一个代理对象

### ProxyTransactionManagementConfiguration

- 一个配置类，定义了另外3个bean
  1. `BeanFactoryTransactionAttributeSourceAdvisor`：一个 Advisor
  2. `AnnotationTransactionAttributeSource`：相当于上面 Advisor 的 pointcut
     1. 用来判断某个类 或 方法上是否存在 @Transactional 注解
  3. `TransactionInterceptor`：相当于上面 Advisor 的 Advice
     1. 代理逻辑，事务代理对象执行方法时，最终会进入到 TransactionInterceptor 的 invoke() 方法中

## 基本执行原理

1. 利用所配置的 `PlatformTransactionManager` 事务管理器，新建一个数据库连接
2. 修改数据库连接的 `autocommit` 为 `false`
3. 执行 `MethodInvocation.proceed()`，即执行业务方法，其中就会执行sql
4. 正常则提交事务，异常则回滚

## 事务详细执行流程图

[Spring事务底层原理流程图](https://www.processon.com/view/link/61b2f2437d9c0843715025b8)

## Spring事务传播机制

### 应用场景

- a() 调用 b()
  1. a() 和 b() 中所有SQL都需要在同一个事务中
  2. a() 和 b() 需要单独的事务
  3. a() 需要在事务中执行，b() 不需要在事务中执行
  4. ...

### 实现原理

- 以 a() 在一个事务中执行，调 b() 需要新开一个事务执行 为例：

1. 首先，代理对象执行 a() 前，先利用 事务管理器 新建一个数据库连接 a
2. 将数据库连接 a 的 autocommit 设置为 false
3. 把数据库连接 a 设置到 ThreadLocal 中
4. 执行 a() 中的 sql
5. 执行 a() 过程中，调用了 b() （注意，用的 代理对象 调用 b()）
   1. **核心：代理对象执行 b() 前，判断出当前线程中已经存在一个 数据库连接 a ，表示当前线程已经有一个 Spring 事务了，则进行 `挂起`**
   2. `挂起` 就是将 ThreadLocal 中的数据库连接 a 移除，并放入一个 `挂起资源对象` 中
   3. 挂起完成后，再利用 事务管理器 新建一个 数据库连接 b
   4. 将 b 的 autocommit 设置为 false
   5. 将 b 设置到 ThreadLocal 中
   6. 执行 b() 中的 sql
   7. b() 正常执行完，则从 ThreadLocal 中拿到 b 连接并进行提交
   8. 提交后，恢复挂起的数据库连接 a，就是从 `挂起资源对象` 中取出 a 再次设置到 ThreadLocal 中
6. a() 正常执行完，则从 ThreadLocal 中拿到 a 进行提交

## Spring事务传播机制分类

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1639119395325.png)

- 以非事务方式运行：
  - 表示执行该方法时，Spring事务管理器不会建立数据库连接
  - 由 ORM 框架自己建立数据库连接来执行 sql

## 案例分析

### 案例1

```java
@Component
public class UserService {
	@Autowired
	private UserService userService;

	@Transactional
	public void test() {
		// test方法中的sql
		userService.a();
	}

	@Transactional
	public void a() {
		// a方法中的sql
	}
}
```

- 默认情况下，传播机制为 `REQUIRED` 
  - 表示当前如果没有事务，则新建一个事务
  - 如果有事务，则在当前事务中执行
- 所以上面代码执行流程如下：
  1. 新建一个数据库连接 conn
  2. 设置 conn 的 autocommit 为 false
  3. 执行 test 方法中的 sql
  4. 执行 a 方法中的 sql
  5. 执行 conn 的 commit() 进行提交

### 案例2

```java
@Component
public class UserService {
	@Autowired
	private UserService userService;

	@Transactional
	public void test() {
		// test方法中的sql
		userService.a();
        int result = 100/0;
	}

	@Transactional
	public void a() {
		// a方法中的sql
	}
}
```

1. 新建一个数据库连接 conn
2. 设置 conn 的 autocommit 为 false
3. 执行 test() 中的 sql
4. 执行 a() 中的 sql
5. 抛出异常
6. 执行 conn 的 rollback() 进行回滚，两个方法的 sql 都会被回滚

### 案例3

```java
@Component
public class UserService {
	@Autowired
	private UserService userService;

	@Transactional
	public void test() {
		// test方法中的sql
		userService.a();
	}

	@Transactional(propagation = Propagation.REQUIRES_NEW)
	public void a() {
		// a方法中的sql
		int result = 100/0;
	}
}
```

1. 新建一个数据库连接 conn
2. 新建 conn 的 autocommit 为 false
3. 执行 test 方法中的 sql
4. 新建一个数据库连接 conn2
5. 执行 a() 中的 sql
6. 抛出异常
7. 执行 conn2 的 rollback() 回滚
8. 继续抛异常，test() 调 a() 接收异常，然后抛出
9. 执行 conn 的 rollback() 回滚，最终还是两个方法的 sql 都回滚

## Spring事务强制回滚

### 场景

- a() 调 b()，传递事务，b() 异常，a() catch 住
- 但 a() catch 是为了给用户友好提示，还想继续 rollback

### 案例

```java
@Transactional
public void test(){
	
    // 执行sql
	try {
		b();
	} catch (Exception e) {
		// 构造友好的错误信息返回
		TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
	}
    
}

public void b() throws Exception {
	throw new Exception();
}
```

## TransactionSynchronization

- Spring事务状态可能为 提交、回滚、挂起、恢复
- 可以用如下机制，监听当前 Spring 事务所处状态

```java
@Component
public class UserService {

	@Autowired
	private JdbcTemplate jdbcTemplate;

	@Autowired
	private UserService userService;

	@Transactional
	public void test(){
		TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {

			@Override
			public void suspend() {
				System.out.println("test被挂起了");
			}

			@Override
			public void resume() {
				System.out.println("test被恢复了");
			}

			@Override
			public void beforeCommit(boolean readOnly) {
				System.out.println("test准备要提交了");
			}

			@Override
			public void beforeCompletion() {
				System.out.println("test准备要提交或回滚了");
			}

			@Override
			public void afterCommit() {
				System.out.println("test提交成功了");
			}

			@Override
			public void afterCompletion(int status) {
				System.out.println("test提交或回滚成功了");
			}
		});

		jdbcTemplate.execute("insert into t1 values(1,1,1,1,'1')");
		System.out.println("test");
		userService.a();
	}

	@Transactional(propagation = Propagation.REQUIRES_NEW)
	public void a(){
		TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {

			@Override
			public void suspend() {
				System.out.println("a被挂起了");
			}

			@Override
			public void resume() {
				System.out.println("a被恢复了");
			}

			@Override
			public void beforeCommit(boolean readOnly) {
				System.out.println("a准备要提交了");
			}

			@Override
			public void beforeCompletion() {
				System.out.println("a准备要提交或回滚了");
			}

			@Override
			public void afterCommit() {
				System.out.println("a提交成功了");
			}

			@Override
			public void afterCompletion(int status) {
				System.out.println("a提交或回滚成功了");
			}
		});

		jdbcTemplate.execute("insert into t1 values(2,2,2,2,'2')");
		System.out.println("a");
	}


}
```

