# 杨过 - Synchronized 详解

## 设计同步器的意义

```shell
# 在多线程编程中，可能会出现 多个线程 同时访问 同一个 共享、可变资源 的情况，
	# 这种资源 称之为 临界资源，
	
	# 这种资源可能是 对象、变量、文件...
	
# 共享: 资源可以由 多个线程 同时访问。

# 可变: 资源可以在其 生命周期 内被修改。
```

```shell
# 引出的问题:
	# 由于 线程执行过程 是不可控的，
	
	# 所以需要采用 同步机制 来协同 对象可变状态 的访问。
```

### 如何解决线程并发安全问题

```shell
# 实际上，所有的 并发模式 在解决 线程安全问题时，
	# 都是使用 序列化访问临界资源，
	
	# 即: 同一时刻，只能有一个线程访问 临界资源。
	
	# 也称作 同步互斥访问。
```

```shell
# Java 中，提供了两种方式来实现 同步互斥访问，
	# 即: synchronized 和 lock
```

```shell
# 同步器的本质就是 加锁。
	# 加锁目的: 序列化访问临界资源。
	
# 注意:
	# 当 多个线程 执行 同一个方法 时，方法内部的局部变量 并不是 临界资源，
	
	# 因为 局部变量 是在 每个线程的私有栈 中，不具有 共享性，也不会导致 线程安全问题。
```

## Synchronized 原理详解

```shell
# synchronized 内置锁 是一种 对象锁（锁的是对象，而非引用），
	# 作用粒度是 对象，
	
	# 可以实现 临界资源的同步互斥访问，
	
	# 是 可重入锁。
```

```shell
# 加锁方式:

1. 同步实例方法，锁当前实例对象

2. 同步类方法，锁当前类对象

3. 同步代码块，锁括号里面的对象
```

### Synchronized 底层原理

```shell
# synchronized 基于 JVM内置锁 实现，通过 内部对象Monitor（监视器锁）实现。

# 基于 进入与退出 Monitor 对象，实现 方法与代码块 同步，
	# Monitor 的实现 依赖于 底层操作系统的 Mutex lock（互斥锁）实现，
	
	# 这是一个 重量级锁，性能较低。
	
# JVM内置锁 在 1.5版本后 做了 重大优化，
	# 如: 
		# 锁粗化（Lock Coarsening）
		# 锁消除（Lock Elimination）
		# 轻量级锁（Lightweight Locking）
		# 偏向锁（Biased Locking）
		# 适应性自旋（Adaptive Spinning）
		# ...
		
	# 使用 上述技术 来减少 锁操作的开销，
	
	# JDK1.5 之后，内置锁的并发性能 已经基本与 Lock 持平。
```

```shell
# synchronized 关键字被 编译成字节码 后，会被翻译成如下两条指令:
	# monitor enter: 位于 同步块逻辑代码 的 起始位置
	
	# monitor exit: 位于 同步块逻辑代码的 的 结束位置
	
# 注意:
	# monitor enter（实际指令是 monitorenter）
	
	# 这里写分开是增加可读性，monitor exit 同理。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603695148519.png)

```shell
# 每个 同步对象 都有一个自己的 Monitor（监视器锁），加锁过程如下图所示:
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603695198342.png)

### Monitor 监视器锁

```shell
# 任何一个对象 都有一个 Monitor 与之关联，
	# 当且仅当一个 Monitor 被持有后，该对象将处于 锁定状态。
	
# synchronized 在 JVM 里的实现，
	# 都是通过 成对 的 monitor enter 和 monitor exit 指令来实现。
```

#### monitor enter

```shell
 # monitor enter:
 	# 当 monitor 被占用时，就会处于 锁定状态，
 	
 	# 线程执行 monitor enter 指令时，尝试获取 monitor 的所有权，
 	
# 过程如下:
 	# 如果 monitor 的进入数为 0，则该线程进入 monitor，设置进入数为1，线程持有该 monitor。
 	
 	# 如果线程已经持有该 monitor，只是重新进入，则该 monitor 的 进入数加1
 	
 	# 如果其他线程已经持有该 monitor，则该线程进入 阻塞状态，
 		# 直到 monitor 的进入数为 0，再重新尝试获取 monitor 的所有权。
```

#### monitor exit

```shell
# monitor exit:
	# 执行 monitor exit 的线程必须是 monitor 的持有者。
	
	# 指令执行时，monitor 的进入数减1，如果 减1后为0，线程退出 monitor，
	
	# 其他被 monitor 阻塞的线程 尝试获取该 monitor。
	
# monitor exit 指令有两次:
1. 同步正常退出，释放锁

2. 发生异常退出，释放锁
```

#### Java 中的 synchronized

```shell
# synchronized 的语义是通过 monitor 对象完成。

# wait/notify 等方法 也依赖于 monitor 对象，
	# 只有在 同步代码块、同步方法 中才能调用 wait/notify 等方法，
	
	# 否则会抛出 IllegalMonitorStateException 
```

#### ACC_SYNCHRONIZED

```java
// 举例: 同步方法的反编译结果
public class SynchronizedMethod {
  
  public synchronized void method() {
    System.out.println("Hello World!");
  }
    
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603701786413.png)

```shell
# 从编译结果来看，同步方法 并没有 通过指令 monitor enter 和 monitor exit 来完成，
	# 相对于 普通方法，其 常量池 中多了 ACC_SYNCHRONIZED 标识符，
	
	# JVM 即利用 该标识符 ACC_SYNCHRONIZED 来实现 方法的同步。

# 当方法调用时，调用指令 将会检查方法的 ACC_SYNCHRONIZED 访问标识 是否被设置，
	# 如果设置了，执行线程 将先获取持有 monitor，
	
	# 获取成功再执行 方法体，方法执行完后再释放 monitor，
	
	# 方法调用期间，其他线程都无法再 获取持有 同一个 monitor 对象。
```

```shell
# 直接 monitor enter、exit 和 设置 ACC_SYNCHRONIZED 没有本质区别，
	# 只是 ACC_SYNCHRONIZED 隐式罢了。
```

#### 缺陷

```shell
# monitor entry、exit 
```



### 什么是 Monitor？

## 对象的内存布局

### 对象头

### 对象头分析工具

### 锁的膨胀升级过程

#### 偏向锁

#### 轻量级锁

#### 自旋锁

#### 锁消除

#### 逃逸分析