[TOC]

# 杨过 - CPU缓存一致性协议MESI

## JVM-JMM-CPU底层全执行流程

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603682815257.png)

## volatile 通过 MESI 实现可见性

```shell
# volatile 就是通过 MESI 实现可见性。

# volatile 修饰变量，在进行赋值操作时，可查看其 汇编指令 原语:
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603683274224.png)

### lock 在汇编中的意义

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603683313521.png)

```shell
# 在 修改内存操作 时，使用 LOCK 前缀去调用 加锁的 读-修改-写 操作（原子的）。
	# 这种机制 用于 多线程系统中 处理器之间进行可靠通讯。
	
# 在 Pentium(奔腾处理器) 和 早期的 IS-32 处理器中，
	# LOCK 前缀会使 处理器执行当前指令时，产生一个 LOCK# 信号，
	
	# 这总是引起 显式总线锁定 出现，
	
	# 该处理器就在 锁定操作期间 不会响应总线控制请求。
	
# 这是由于 早期技术 较为落后，CPU 还没有三级缓存，
	# 可能只有 一级缓存，或者直接 寄存器 与 主存bus总线 进行交互，
	
	# 这时候 lock 语义实际上还是上锁，在 主存bus总线上锁，
	
	# 防止 多核cpu 同时对共享变量进行赋值 可能出现的问题。
	
# 所以早期 CPU 性能是较为低下的，因为一旦触发 总线锁 机制，
	# 多核CPU 实际上就变为了 单核CPU。
```

```shell
# 基于上述，引出 MESI 多核CPU缓存一致性协议。
```

## MESI

```shell
# 多核CPU 的情况下，存在多个 一级缓存，
	# 如何保证 缓存内部数据 一致？
	
	# 这就引出了 一致性协议 MESI
	
# 多核CPU 采用监听机制，监听 主存bus总线，
	# 共享变量的更改都会被感知到。
```

### MESI协议缓存状态

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603684197118.png)

```shell
# MESI 是指 4种状态 的首字母。

# 每个 Cache line（缓存行，缓存存储数据的单元） 有4个状态，可以用 2个bit 表示。
```

| 状态                   | 描述                                                         | 监听任务                             |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------ |
| M（修改Modified）      | 该Cache line有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache 中 | 缓存行必须时刻监听所有视图读该缓存行 |
| E（独享互斥Exclusive） | 该Cache line有效，数据和内存中的数据一致，数据只存在于本Cache 中。 |                                      |
| S（共享Shared）        | 该Cache line有效，数据和内存中的数据一致，数据存在于很多Cache中。 |                                      |
| I（无效Invalid）       | 该Cache line无效。                                           | 无                                   |

#### 注意

```shell
# 对于 M 和 E 状态而言 总是精确的, 在和该 缓存行 的真正状态是一致的,
	# 而 S 状态可能是 非一致的。
	
# 如果一个缓存将 处于S状态 的缓存行作废了，
	# 而另一个缓存实际上可能已经 独享 了该缓存行，
	
	# 但该 缓存 却不会将该 缓存行 变更为 E 状态，
	
	# 因为 其他缓存 不会广播 他们作废掉该缓存行的通知，
	
	# 同样，由于缓存并没有保证 该缓存行的 copy 数量，

	# 因此（即时有这种通知），也无法确定 自己 是否已经 独享该缓存行。
	
# E 状态是一种 投机性 的优化:
	# 如果一个 CPU 想修改一个处于 S状态 的缓存行，
	
	# 总线事务 需要将所有 该缓存行的copy 变成 invalid 状态，
	
	# 而修改 E 状态的缓存 不需要使用 总线事务。
```

### MESI状态转换

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603090984813.png)

#### 触发事件

| 触发事件                 | 描述                       |
| ------------------------ | -------------------------- |
| 本地读取（Local read）   | 本地cache读取本地cache数据 |
| 本地写入（Local write）  | 本地cache写入本地cache数据 |
| 远端读取（Remote read）  | 其他cache读取本地cache数据 |
| 远端写入（Remote write） | 其他cache写入本地cache数据 |

#### cache分类

```shell
# 前提: 所有的cache共同缓存了主内存中的某一条数据

# 本地cache: 指 当前cpu 的 cache

# 触发cache: 触发读写事件的 cache

# 其他cache

# 注意: 本地的事件触发 本地cache 和 触发cache 为相同
```

#### 上图切换解释

| 状态 | 触发本地读取                                                 | 触发本地写入                                                 | 触发远端读取                                             |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------------- |
| M    | 本地cache:M<br>触发cache:M<br>其他cache:I                    | 本地cache:M<br/>触发cache:M<br/>其他cache:I                  | 本地cache: M->E->S<br>触发cache: I->S<br>其他cache: I->S |
| E    | 本地cache:E<br>触发cache:E<br>其他cache:I                    | 本地cache: E->M<br>触发cache: E->M<br>其他cache: I           | 本地cache: E->S<br>触发cache: I->S<br>其他cache: I->S    |
| S    | 本地cache:S<br>触发cache:S<br>其他cache:S                    | 本地cache: S->E->M<br>触发cache: S->E->M<br>其他cache: S->I  | 本地cache:S<br>触发cache:S<br>其他cache:S                |
| I    | 本地cache: I->S  或 I->E<br>触发cache: I->S 或 I->E<br>其他cache: E、M、I->S、I | 本地cache: I->S->E->M<br>触发cache: I->S->E->M<br>其他cache: M、E、S->S、I | 既然本cache是I<br>其他cache操作与本cache无关             |

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603095563364.png)

```shell
# 举例:
	# 假设 cache1 中 a = 1 变量的cache line 处于 S，
	
	# 其他拥有 a 变量的 cache 中的 cache line 调整为 S 或 I
```

## 多核缓存协同操作

```shell
# 假设现有三个CPU，A、B、C， 对应三个缓存分别是cache a、b、c，主内定义 x = 0。
```

### 单核读取

```shell
# CPU A 发出指令，从主存中读取 x
	# -> 从主存通过 bus 读取到缓存中（远端读取 Remote read）
	# -> 此时该 cache line 修改为 E（独享）
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603095860316.png)

### 双核读取

```shell
# CPU A 发出指令，从主存中读取 x
	# -> 从主存通过 bus 读取到 cache a 中，并将 cache line 设置为 E
	
# CPU B 发出指令，从主存中读取 x
	# 此时，CPU A 检测到地址冲突，对相关数据做出响应，
	
	# 此时，x 存储于 cache a 和 cache b 中，都被设置为 S（共享）
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603096076444.png)

### 修改数据

```shell
# CPU A 计算完成后，发布指令，需要修改 x，
	# CPU A 将 x 设置为 M（修改），并通知 缓存了x 的 CPU B，
	
	# CPU B 将 cache b 中的 x 设置为 I（无效），
	
	# CPU A 对 x 进行赋值
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603096189207.png)

### 同步数据

```shell
# CPU B 发出指令，读取 x，通知 CPU A
	# CPU A 将 修改后的数据 同步到主内存时，cache a 修改为 E，
	
	# CPU A 将 x 同步给 CPU B，将 cache a 和 同步后 cache b 的 x 设置为 S。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603096344264.png)

## 缓存行伪共享

### 什么是伪共享

```shell
# CPU 缓存系统中，是以 cache line（64 Bytes）为单位存储的，
	# 多线程情况下，如果需要修改 "共享同一个缓存行的变量"，
	
	# 就会 无意中 影响彼此的性能，这就是 伪共享（False Sharing）
	
# 举例:
	# t1 访问 a，t2 访问 b,
	
	# 恰好 变量 a b 在同一个 cache line 上，
	
	# t1 修改 a，将导致 b 也被刷新。
```

### 怎么解决伪共享

```shell
# jdk8 新增注解 @sun.misc.Contended

# 加上该注解的类，会自动补齐 缓存行，
	# 该注解默认无效，需要开启JVM参数: -XX:-RestrictContended
```

## MESI优化和引入的问题

```shell
# CPU 缓存的一致性消息传递，是需要时间的，
	# 这会造成 CPU 的阻塞，因为 等待其他CPU Cache回应时间 远远大于 执行指令时间。
```

### 存储缓存（Store Bufferes）

```shell
# 为避免这种 CPU运算能力的浪费，引入 Store Buffer。

# CPU 把想要 写入主存的值 写到缓存，然后继续去处理其他指令，
	# 当所有失效（invalid ack）都接收到时，数据才会最终提交。
```

#### Store Bufferes的风险

```shell
# 1. CPU 会尝试从 Store buffer 中读取值，但它还没有进行提交。

# 解决方案:
	# Store Forwarding，加载的时候，如果 Store buffer 中存在，则进行返回。
```

```shell
# 2. 保存什么时候完成？这个并没有任何保证。
```

#### 硬件内存模型

```shell
# 执行失效 也需要 CPU 处理，
	# Store buffer 容量也是有限的，
	
	# 所以 CPU 有时需要等待 失效确认 的返回，这会让 性能 大幅降低。
	
# 为了解决这种问题，引入 失效队列。
```

```shell
# 失效队列:
	# 1. 对于所有收到的 invalid 请求，invalid ack 消息必须立刻发送。
	
	# 2. invalid 并不真正执行，而是放在一个 特殊队列 中，在方便的时候才会去执行。
	
	# 3. CPU 不会发送任何消息给所处理的 cache line，直到它处理 invalid。
```

