[TOC]

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
# monitor entry、exit 两个指令的执行，
	# 是 JVM 通过调用 操作系统的互斥原语(mutex) 来实现，
	
	# 被阻塞的线程会被挂起，等待重新调度，
	
	# 会导致 "用户态和内核态" 之间来回切换，对性能有较大影响。
```

### 什么是 Monitor？

```shell
# Monitor 可以理解为 一个同步工具，也可以描述为 一种同步机制，通常被描述为 一个对象。

# 所有 Java 对象都是天生的 Monitor，
	# 在 Java 的设计中，每个 Java 对象都自带一把 内部锁(Monitor锁)，
	
	# 即通常所说 Synchronized 的对象锁，
	
	# MarkWord 锁标识位 为 10，其中指针指向 Monitor 对象的 起始地址。
```

#### HotSpot 源码

```shell
# HotSpot 中，Monitor 是由 ObjectMonitor 实现的。

# C++ # HotSpot # JVM源码 # ObjectMonitor.hpp 文件 
```

```c++
ObjectMonitor() {
  _header = NULL;
  _count = 0; // 记录个数
  _waiters = 0;
  _recursions = 0;
  _object = NULL;
  _owner = NULL;
  _WaitSet = NULL; // 处于 wait 状态的线程，会被加入到 _WaitSet
  _WaitSetLock = 0;
  _Responsible = NULL;
  _succ = NULL;
  _cxq = NULL;
  FreeNext = NULL;
  _EntryList = NULL; // 处于等待锁 block 状态的线程，会被加入该列表
  _SpinFreq = 0;
  _SpinClock = 0;
  OwnerIsThread = 0;
}
```

```shell
# ObjectMonitor 有两个队列:
	# _WaitSet 
	
	# _EntryList
	
# 这两个队列用来保存 ObjectWaiter 对象列表（每个等待锁的线程，都会被封装成ObjectWaiter对象）

# _owner 指向持有 ObjectMonitor 对象的线程。
```

#### Monitor 工作原理

```shell
# 当多个线程同时访问一段同步代码时:

# 1. 首先进入 _EntryList 集合，
	# 当 线程 获取到对象的 Monitor 后，
	
	# 进入 _owner 区域，并把 Monitor 中的 owner 变量设置为 当前线程，
	
	# 同时 monitor 中的计数器 count++。
	
# 2. 若线程调用 wait() 方法，将释放当前持有的 Monitor，
	# owner 变量恢复为 null，count--，
	
	# 同时 该线程进入 WaitSet 集合中等待被唤醒。
	
# 3. 若当前线程执行完毕，也将释放 Monitor 并复位 count 的值，以便其他线程进入获取 Monitor。
```

#### Monitor 与 对象头

```shell
# Monitor 对象存在于每个 Java 对象的对象头（Mark Word）中 存储的指针的指向，
	# Synchronized 便是通过这种方式来获取锁，
	
	# 也是 Java 所有对象皆为 锁 的原因，
	
	# 同时 notify/notifyAll/wait 等方法，都会使用到 Monitor 锁对象，所以必须在同步代码块中使用。
```

```shell
# Monitor 有两种同步方式:
	# 互斥
	
	# 协作
	
# 多线程环境下，线程之间如果需要共享数据，需要解决 互斥访问数据 的问题，
	# Monitor 可以确保 Monitor锁对象上的数据 在同一时刻只会有一个线程在访问。
	
# Synchronized 加锁在对象上，锁状态 是被记录在每个对象的对象头（Mark Word）中。
```

## 对象的内存布局

```shell
# HotSpot JVM 中，对象在内存中存储的布局分为三块区域:

# 对象头（Header）
	# 主要存储如下数据:
    # hash code

    # 对象所属年代

    # 对象锁

    # 锁状态标志

    # 偏向锁（线程）ID

    # 偏向时间

    # 数组长度（数组对象） 
    
    # ...

# 实例数据（Instance Data）
	# 存放类的属性数据信息，包括父类的属性信息

# 对齐填充（Padding）
	# 因为 JVM 要求，对象起始地址必须是 8字节 的整数倍，
	
	# 填充数据仅仅是为了字节对齐
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605495423153.png)

### 对象头

```shell
# Java对象头一般占有 2个机器码，
	# 32位JVM中，1个机器码等于4字节，即 32bit
	
	# 64位JVM中，1个机器码等于8字节，即 64bit
	
# 但如果 对象 是 数组类型，则需要 3个机器码，
	# 因为 JVM 可以通过 Java对象的元数据信息 确定 Java对象的大小，
	
	# 但无法从数组的元数据来确认数组的大小，因此需要额外 1个机器码 记录数组长度。
```

```shell
# 关于对象头笔记，参考 性能调优专题 JVM。

# 对象头中，第一部分 Mark Word 存储对象自身的运行时数据，如 hash code...
	# Mark Work 是实现 轻量级锁 和 偏向锁 的关键。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605496210658.png)

### 锁的膨胀升级过程

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605513189369.png)

```shell
# 锁的状态总共有 4种:
	# 无锁状态
	
	# 偏向锁
	
	# 轻量级锁
	
	# 重量级锁
```

```shell
# 随着 锁 的竞争，锁可以从 偏向锁 升级至 轻量级锁 升级至 重量级锁。

# 锁的升级是单向的，只能从低到高，不会出现锁的降级。

# JDK1.6后，默认开启 偏向锁 和 轻量级锁。
```

#### 偏向锁

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605516515925.png)

```shell
注意: 服务器分大小端模式，我们使用默认都是小端模式，把上述 32bit 倒置过来看就好。

Java 初始对象是一个 匿名偏向 的状态，准备好被 偏向线程 持有。
```

```shell
# JDK1.6 之后加入的新锁，是一种 针对加锁操作的优化手段。

# 大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，
	# 因此为了减少 同一线程获取锁（涉及CAS操作，耗时）的代价，引入 偏向锁。
```

```shell
# 偏向锁的核心思想:
	# 如果 一个线程 获得了锁，那么 锁 就进入 偏向模式，
	
	# 此时 Mark Word 的结构也变为 偏向锁结构，
	
	# 当 该线程 再次请求 锁 时，无需做任何 同步操作，就获取锁。
	
	# 这样就省去了大量有关 锁申请 的开销，提高程序性能。
```

```shell
# 对于没有 锁竞争 的场合，偏向锁 具有很好的 优化效果。

# 但是对于 锁竞争 比较激烈的场合，偏向锁就失效了，会升级为 轻量级锁。
```

#### 轻量级锁

```shell
# 偏向锁失效后，升级为 轻量级锁，此时 Mark Word 的结构也变为 轻量级锁 的结构。

# 轻量级锁的核心思想:
	# 对绝大部分的锁，在整个 同步周期内 都不存在竞争。
	
# 轻量级锁 适应的场景是:
	# 线程交替 执行同步块。
	
# 如果存在 同一时间 访问 同一锁 的场合，轻量级锁失效，升级为 重量级锁。
```

#### 自旋锁

```shell
# 轻量级锁 失效后，JVM为了避免 线程挂起，会进行 自旋锁 的优化。

# 大部分情况下，线程持有锁的时间都不会太长，
	# 如果 阻塞线程 直接挂起，会造成 CPU线程上下文切换 的性能开销。（用户态、内核态的切换）
	
	# 所以，JVM 会让线程先做几个空循环（自旋），一段自旋后，可能就得到锁，获取到临界资源了。
	
	# 如果 一段字段 后，还没有获取锁，就只能将 线程挂起。
	
# 这就是 自旋锁的优化方式，如果还是不行，就只能升级为 重量级锁 了。
```

#### 锁消除

```shell
# JVM 在 JIT即时编译时（某段代码即将第一次被执行时，进行编译），
	# 通过对 运行上下文 的扫描，去除不可能存在 共享资源竞争 的锁，从而节省 毫无意义请求锁 的时间。
	
# 锁消除 依托于 逃逸分析 的数据支持。
```

#### 锁粗化

```java
/**
 * 对于如下代码，JVM会根据逃逸分析，优化成如下代码:
 * 	 synchronized(lock) {
 *     System.out.println("第一个同步块");
 *		 System.out.println("第二个同步块");
 *		 System.out.println("第三个同步块");
 *   }
 *
 * 粗化锁的范围，减少无意义的 获取锁、解除锁 的性能开销。
 */
public class Demo {
  
  public static final Object lock = new Object();
  
  public static void main(String[] args) {
    synchronized(lock) {
      System.out.println("第一个同步块");
    }
    
    synchronized(lock) {
      System.out.println("第二个同步块");
    }
    
    synchronized(lock) {
      System.out.println("第三个同步块");
    }
  }
  
}
```



#### 逃逸分析

```shell
# 关于逃逸分析的介绍同样参考 性能调优专题 - JVM部分。

# 使用逃逸分析，编译器可以对代码做如下优化:

1. 同步省略。
	# 如果一个对象被发现只能从 一个线程 中被访问到，那对 该对象 的操作，无需考虑同步问题。
	
2. 将堆分配转换为栈分配。
	# 如果一个对象，在子线程中被分配，要使指向该对象的指针永远不会逃逸，
	
	# 对象可能是 栈分配 的候选，而不是堆分配。
	
3. 分离对象或标量替换。
	# 有的对象可能不需要 一个连续的内存空间 存储，那么可以拆分存放在 CPU寄存器中。
```

