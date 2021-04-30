[TOC]

# 诸葛 - G1&ZGC

## G1 收集器

### G1 简介

```shell
# G1 : Garbage-First
	# 主要针对 多核CPU 及 大容量内存 的机器,

	# 以 极高的概率 满足 GC STW 时间要求的同时，
	
	# 还具备 高吞吐量 性能特征。
```

### G1 特点

```shell
# G1 是 JDK1.7 之后 JVM 的一个重要进化特征（自 JDK1.7之后提供 G1），具备以下特点:
	#（这里估计就是鞭尸 Serial Parallel，没什么值得理解记忆的必要性。）
```

```shell
# 并行与并发
	# G1 能充分利用 多核CPU 的硬件优势，
	
	# 使用 多核CPU 来缩短 STW 时间。
	
	# 部分其他收集器原本需要 STW 来执行 GC 动作
	
	# 而 G1 仍然可以通过并发的方式、让 用户线程 与 GC线程 同时工作。
```

```shell
# 分代收集
	# 虽然 G1 可以独立管理整个 堆，但是还是保留了 分代的概念。
```

```shell
# 空间整合
	# 与 CMS 标记清除 算法不同，
	
	# G1 从整体上来看，是基于 标记整理 算法实现的 GC 收集器，
	
	# 但从局部细微来看，G1 本质是基于 复制算法 实现的 GC 收集器。
```

```shell
# 可控制的 STW
	# G1 相对于 CMS 的另一个大优势，
	
	# 降低停顿时间是 G1 和 CMS 共同的关注点，
	
	# 但 G1 除了追求低停顿外，还能建立 可预测的停顿时间模型，
	
	# 能让使用者明确指定，在一个长度为 X 毫秒的时间片段内完成 GC。
```

### G1 概览图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596528900096.png)

### G1 详细介绍

```shell
# G1 将 JVM 堆 划分成 多个大小相等 的独立区域（Region），
	# JVM 最多可以有 2048 个 Region。
	
# 一般 Region 大小 计算公式:
	# 堆总大小 / 2048
	
	# 比如 堆总大小 4096M，则每个 Region 大小为 2M
	
	# 可以使用参数 -XX:G1HeapRegionSize 手动指定 Region 大小，但是推荐默认计算方式。
```

```shell
# G1 保留了 年轻代 和 老年代 的概念，
	# 但不再是通过 物理空间 进行 隔离 了，
	
	# 它们都是（可以不连续）Region 的集合。
```

```shell
# 默认 年轻代 对 堆内存 的占比是 5%，
	# 比如: 堆总大小 4096M，则 年轻代 占据 200M 左右的内存空间，对应大概是 100个 Region，
	
	# 可以通过 -XX:G1NewSizePercent 设置 新生代初始占比。
	
# 在实际项目运行的过程中，JVM 会不停的 动态调整 年轻代的 Region 占比，
	# 但是最多 年轻代的占比 不能超过 堆总大小的 60%，
	
	# 可以通过 -XX:G1MaxNewSizePercent 调整 年轻代 的最大占比。
	
# 年轻代中的 Eden 和 Survivor 比例和之前保持一致，默认 8:1:1。
	# 比如: 年轻代现在有 100 个Region，Eden 区对应 80 个，s0 对应 10个，s1 对应 10个。
```

```shell
# 一个 Region 所属分区，是会动态变化的，
	# 比如: Region 开始是年轻代，在对其进行了 GC 后，可能会变成 老年代。
```

```shell
# G1 收集器对于 对象 什么时候进入到 老年代，跟之前的规则基本一致，
	# 唯一不同的，是 G1 对 大对象的处理。
	
	# G1 有专门分配 大对象的 Region 叫 Humongous 区，
		# 而不是让 大对象 直接进入 老年代 的 Region 区。
		
# 在 G1 中，大对象的判定规则就是:
	# 对象大小 超过一个 Region 大小的 50%，
	
	# 比如: 每个 Region 大小是 2M，那么只要对象大小超过了 1M，就会被放入 Humongous 中，
	
	# 如果 对象大小超过了 2M，G1 会让其横跨多个连续的 Humongous Region 存放。
	
# Humongous Region 专门用来存放 巨型对象，
	# 节约了 老年代 的空间，可以让 老年代 存放 更多的系统存活时间长 的对象，
	
	# 减少了 系统对象 进入老年代、老年代空间可能不够的 Full GC 开销。
	
# G1 Mixed GC 和 Full GC 回收对象时，会收集 年轻代、老年代、元空间 以及 Humongous。
```

### G1 收集流程

#### 初始标记

```shell
# 初始标记 : initial mark, STW

# 暂停所有用户线程，并记录下 GC Roots 能够直接引用的对象，速度很快。
```

#### 并发标记

```shell
# 并发标记 : Concurrent Marking

# 同 CMS 的并发标记。
```

#### 最终标记

```shell
# 最终标记 : Remark, STW

# 同 CMS 的重新标记。
```

#### 筛选回收

```shell
# 筛选回收 : CleanUp，STW

# 该阶段首先对各个 Region 的 回收价值 和 成本 进行排序，
	# 根据 开发 所期望的 STW 时间（-XX:MaxGCPauseMillis）来制定回收计划，
	
	# 比如说: 老年代此时有 1000 个 Region 都满了，
	
	# 但是 预期停顿时间 设置为 200毫秒，
	
	# 那么通过之前 回收成本 计算得知，可能回收其中 800个 Region 刚好需要 200ms，
	
	# 那么 G1 就只会回收这 800个 Region（Collection Set，要回收的集合），
	
	# 尽量把 GC STW 的时间，控制在 我们指定的时间内。
	
	
# 这个阶段，其实可以做到与 用户线程 一起并发执行，
	# 但是因为 只回收一部分 Region，时间还得是 开发 可控制的，
	
	# 而且 STW 将大幅提高 收集效率，
	
	# 所以 G1 将其做成了 STW 的阶段。
	
	# 小说明:
		# CMS 回收阶段是与用户线程并行执行的，
		# 但是 G1 内部实现更加复杂，暂时还没有实现 并发回收，
		# 但是有些 GC 收集器已经优化了这一步，
		# 比如 Shenandoah (可以看成 G1 的升级版)
```

```shell
# G1 怎么实现 GC 成本计算的？
	# G1 在后台维护了一个 优先列表，
	
	# 每次根据允许的收集时间，优先选择回收价值最大的 Region
		# 这也是 G1 名字的由来，Garbage-First
		
	# 比如: 一个 Region 100M，经过 GC Roots 标记、存活对象只占 20M
		# 那么 G1 计算、只需复制 20M 对象 即可 清理出一个 100M 的 Region，
   		# 假设这个时间就是 20ms，
    
    # 那么与之相对的，一个 Region 100M，经过 GC Roots 标记，存活对象占了 80M
    	# 那么 G1 计算、需要复制 80M 对象，才能清理出一个 100M 的 Region，
    	# 那么这个时间就是 80ms。
    	
   # 那 G1 在有限时间内，会优先回收哪些对象，已经不言而喻。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596530890965.png)

### G1 回收算法

```shell
# G1 不管是 年轻代 或 老年代，
	# 回收算法 主要都是使用 复制算法，
	
	# 将一个 Region 中的存活对象、复制到另一个 Region 中，
	
	# 不会像 CMS 回收完，还有大量的内存碎片、需要再进行一次压缩整理。
```

### G1 STW时间设置

```shell
# G1 STW 时间默认为 200ms，
	# 一般来说，使用默认即可，
	
	# 如果异想天开的说，设置为 10ms，
	
	# 很可能出现 GC 释放空间大小 < 用户线程产生的新对象大小，
	
	# 持续几次之后，整个堆空间被占满，就会触发 G1的 Full GC，
	
	# 即 回归 Serial 单线程 GC、STW 整个应用。
```

### G1 GC类型

#### Young GC

```shell
# G1 的 Young GC 并不是说，现有的 Eden 放满了就会马上触发，
	# 而是 G1 会计算当前 Eden 回收大概需要多久时间，
	
	# 如果所需 回收时间 远远小于 -XX:MaxGCPauseMills 设定的值，
	
	# 那 G1 会增加年轻代的 Region，用来存放新对象，而不会马上做 Young GC。
	
	# 直到下一次 Eden 被占满，G1 计算 Eden 回收所需时间 接近 -XX:MaxGCPauseMills 设定的值
	
	# G1 才会触发 Young GC。
```

#### Mixed GC

```shell
# G1 中新增的 GC类型，不是 Full GC，
	# 当 老年代 的 堆使用率 达到参数 -XX:InitiatingHeapOccupancyPeercent 设定的值，
	
	# G1 则会触发 Mixed GC，
	
	# 回收所有的 Young、部分Old、Humongous
		# G1 会根据 期望GC STW 时间，确定 Old 区 GC 的优先顺序。
		
# 正常情况下，Mixed GC 回收垃圾对象，
	# 将各个 Region 存活对象复制到别的 Region，
	# 清理原先 Region 中所有垃圾对象，
	# 完成 GC。
	
# 但是，当 G1 发现复制过程中，没有足够的空闲 Region 承载此次存活对象复制，
	# G1 将触发一次 Full GC。
```

#### Full GC

```shell
# STW，采用 单线程 进行 标记清除 所有垃圾对象，然后进行压缩整理。
	# 这个过程是非常耗时的。
```

### G1 参数设置

```shell
-XX:+UseG1GC # 使用 G1 收集器

-XX:ParallelGCThreads # 指定 GC 工作的线程数量

-XX:G1HeapRegionSize 
	# 指定分区大小（1M - 32M，且必须是 2 的N次幂）
	# 默认将整个堆划分为 2048 个分区
	
-XX:MaxGCPauseMillis # 期望 STW 时间（默认200ms）

-XX:G1NewSizePercent # 新生代内存初始空间（默认为 堆总大小 的 5%）

-XX:G1MaxNewSizePercent # 新生代内存最大空间占比

-XX:TargetSurvivorRatio 
	# Survivor 区触发 对象动态判断机制 的比例阈值，默认 50%
	# 如果进入 Survivor 中所有对象大小总和超过了 Survivor 区的 50%，
	# 此时会将年龄 (1+2+...+n) n岁及以上的所有对象放入老年代。
	
-XX:MaxTenuringThreshold
	# 最大进入老年代年龄阈值，默认 15
  
-XX:InitiatingHeapOccupancyPercent
 	# 老年代 占用空间 达到 整堆内存阈值 45%，
 	# 则执行 新生代 和 老年代 的混合收集（Mixed GC）
 	# 比如 2048个Region，老年代占用接近 1000 个，就会触发 Mixed GC
 	
-XX:G1MixedGCLiveThresholdPercent
	# Region 存活对象 与 整个Region 的百分比，默认 85%
	# 存活对象总大小 低于这个百分比时，才会回收该 Region
	# 比如 1G 的 Region, 存活对象只有少于 850M，该 Region 才可能在 GC 时被回收。
	
-XX:G1MixedGCCountTarget
	# 在一次 GC 中，指定做几次 筛选回收，默认 8次，
	# 这个意义是: 将整体的 STW 时间拆分为几次 STW，
	# 避免单次 STW 时间过长。
	
-XX:G1HeapWastePercent
	# 一次 GC 回收 Region 的比例阈值，默认 5%，
	# 比如共 1000 个Region，只要 GC 回收出来的空闲 Region 达到 50 个，
	# 本次 GC 就结束了。
```

### G1 适用场景

```shell
# G1 适合大内存的 Java 应用。

# 比如:
	# Kafka 是一款高并发、高吞吐的消息中间件，
	
	# 假设给 Kafka 分配 64G 内存、Eden 区可以分配 40G 左右内存，
	
	# 每秒处理几十万消息，占满 Edeen 区也是很快的，
	
	# 对于几十个 G 的内存回收，可能最快也要几秒钟的时间，
	
	# 显然是不符合 Kafka 性能期望的，
	
	# 所以，可以使用 G1，在期望停顿时间内，根据优先列表回收，
	
	# 比如每次 GC 花费 200ms，回收 5G 左右空间，
	
	# 那么 Kafka 就可以做到 用户几乎无感 的，一边 GC、一遍处理消息。
```

## ZGC

```shell
# ZGC 是一款追求性能的卓越 GC 收集器，
	# 但是目前还不是完全成熟，也不会有什么公司去使用，
	
	# 所以这里略过。
```

## 如何选择垃圾收集器

```shell
# 4G 以下可以使用 parallel

# 4G-8G 可以使用 ParNew + CMS

# 8G 以上可以使用 G1

# 几百G 以上可以用 ZGC
```

## 安全点与安全区域

```shell
# JVM 触发 GC 时，并不是立马挂起所有用户线程，
	# 而是等所有 用户线程 运行到某个 安全点 或 安全区域 时，自己挂起线程 等待 GC。
	
	# 举例: 
		# A 线程正在执行 CAS 原子操作、此时 GC 触发，
		
		# 如果 A 线程直接被挂起，则破坏了其 CAS 原子性，
		
		# 必然是不被允许的。
```

### 安全点位置类型

```shell
# 方法返回之前

# 调用某个方法之后

# 抛出异常的配置

# 循环的末尾
```

### 安全区域

```shell
# 安全点 是对正在 执行的线程 设定的。

# 如果一个线程处于 Sleep 或 中断状态，就不能及时响应 JVM GC 请求，
	# 也就不能运行到某个 安全点 上，再挂起自身线程，等待 GC。
	
# 所以 JVM 引入 安全区域:
	# 在一段代码片段中，引用关系不会发生变化，此片段即 安全区域，GC 可以直接开始。
	
	# 比如: Thread.sleep()
```

### 实现思想

```shell
# 当 GC 需要暂停 用户线程 时，
	# 不直接对 用户线程 操作，
	
	# 而是设置一个标志位（在哪里设置的不太清楚，可能是 主存上 随意开辟一小块空间用来标识之类）
	
	# 当各个用户线程、运行到 某个安全点 或 安全区域时，
	
	# 就会主动去访问该 标识位，如果为真，则主动挂起自身线程。
```

