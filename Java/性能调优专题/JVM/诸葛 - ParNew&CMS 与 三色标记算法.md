[TOC]

# 诸葛 - ParNew&CMS 与 三色标记算法

## 垃圾收集算法

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596508824966.png)

### 分代收集理论

```shell
# JVM 当前都是采用分代收集的理论，
	# 根据 对象存活周期 的不同，将内存分为几块，
	
	# 一般将 JVM 堆分为 新生代 和 老年代，
	
	# 这样就可以根据各个年代对象特点，选择合适的垃圾回收算法。
	
# 在新生代中，
	# 每次 GC 都会有大量对象被回收，
	
	# 所以新生代可以选择复制算法，
	
	# 只需要付出 少量存活对象 的复制成本 + 两份较小的内存空间 即可完成 GC。
	
# 在老年代中，
	# 对象的存活几率是较高的，也没有多余的大空间对 GC 进行分配担保，
	
	# 所以老年代一般都是使用 标记清除 或 标记整理 算法。
	
	# 通常来说，标记清除 或 标记整理 算法的 GC 会比 复制算法 慢 10倍左右。
```

### 标记 - 复制算法

```shell
# 标记 - 复制算法，是一种高效的垃圾回收算法。

# 它需要两份内存大小一致的空间，
	# 每次都只会使用其中一块，
	
	# 当发生 GC 时，就会将通过 GC Roots 标记的 引用链对象 复制到另一块空间，
	
	# 然后将当前空间进行全部清理。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596509333783.png)

### 标记 - 清除算法

```shell
# 通过 GC Roots 标记的 引用链对象，确定存活对象，
	# 统一回收其他所有未被标记过的对象。
	
# 最基础的收集算法，比较简单，但是有两个明显的问题:
	# 如果需要标记的对象太多，效率不高。
	
	# 标记清除后会产生大量不连续的内存碎片。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596509503255.png)

### 标记 - 整理算法

```shell
# 通过 GC Roots 标记的 引用链对象，确定存活对象，
	# 让所有 存活对象 向一端移动（碰到垃圾对象直接覆盖即可），
	
	# 然后直接清理掉 存活对象 边界以外的内存。
```

## 垃圾收集器

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596509708492.png)

```shell
# 垃圾收集算法 是 内存回收的方法论

# 垃圾收集器 是 内存回收方法论 的具体实现
```

```shell
# 目前没有万能的 GC 收集器，
	# 我们需要根据具体应用场景，选择适合项目的垃圾收集器。
```

### Serial 收集器

```shell
-XX:+UseSerialGC -XX:+UseSerialOldGC
```

```shell
# Serial : 串行、单线程

# 是最基本、历史最悠久的 GC 收集器。

# 它只会使用一条 GC 线程去完成 GC 工作，
	# 并且这整个过程中，都是 STW 的，直到这条 GC 线程完成所有 GC 工作。
	
	# STW : Stop the world，暂停所有用户线程。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596509965373.png)

```shell
# JVM GC 的开发设计者知道，每次 GC STW 会给用户带来不良体验，
	# 所以在后续的 GC 收集器设计中，都在想办法减少 GC STW 的时间。
	
# Serial 的优点:
	# 简单高效: 没有线程上下文切换的开销，自然可以获得很高的 GC 效率。
```

```shell
# Serial Old 是 Serial 的老年代版本，同样是一个单线程收集器。

# Serial Old 的两个用途:
	# 1. 用于 JDK1.5 以及以前的版本中与 Parallel Scavenge 收集器搭配使用
	
	# 2. 作为 CMS 收集器的后备方案
```

### Parallel Scavenge 收集器

```shell
-XX:+UseParallelGC -XX:+UseParallelOldGC
```

```shell
# Parallel 收集器其实就是 Serial 收集器的多线程版本，
	# 除了使用 多线程 进行垃圾收集外，
	
	# 其余行为和 Serial 收集器类似
		# 控制参数、收集算法、回收策略...
		
	# 默认的 GC 线程数跟 系统CPU 核数相同，
		# 可以使用 -XX:ParallelGCThreads 指定 GC 线程数，一般不推荐修改
```

```shell
# Parallel Scavenge 收集器关注点是 吞吐量（高效利用 CPU）
	# 所谓吞吐量就是 CPU 中用于运行用户代码的时间 与 CPU 总消耗时间的比值。
	
	# CMS 等收集器的关注点更多是 减少用户线程停顿时间
	
# Parallel Scavenge 收集器提供了很多参数，供使用者找到最合适的停顿时间 或 最大吞吐量，
	# 如果对于其运作不太了解的话，可以将 内存管理优化 交给 JVM 自己完成。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596512010312.png)

```shell
# Parallel Old 是 Parallel Scavenge 的老年代版本，
	# 使用 多线程 和 标记-整理 算法。
	
# 在注重吞吐量以及 CPU 资源的场合，可以优先考虑 Parallel 收集器

# JDK8 默认的垃圾收集器组合 Parallel Scavenge + Parallel Old
```

### ParNew 收集器

```shell
-XX:+UseParNewGC
```

```shell
# ParNew 跟 Parallel 很类似，
	# 区别在于 它可以和 CMS 配合使用。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596512226446.png)

### CMS 收集器

```shell
-XX:+UseConcMarkSweepGC
```

```shell
# CMS : Concurrent Mark Sweep

# CMS 是一种以 获取最短回收停顿时间 为目标的收集器，
	# 是 HotSpot JVM 第一款真正意义上的并发收集器，
	
	# 第一次实现了让 GC 线程与 用户线程 同时工作（基本上）
	
# CMS 是对 标记-清除 算法实现的。
```

#### CMS 运行步骤

##### 初始标记

```shell
# STW（暂停所有用户线程），记录 GC Roots 能直接引用的对象，速度很快。
```

##### 并发标记

```shell
# 并发标记 是从 GC Roots 的直接引用对象、
	# 开始遍历整个对象图的过程。
	
	# 该过程较为耗时，但不需要停顿用户线程，而是让 用户线程 与 GC线程 并行工作

	# 因为 用户线程 的并行工作，
		# 可能导致 初始标记 中已经标记过的 对象状态 发生改变
```

##### 重新标记

```shell
# 重新标记 是为了修正 并发标记 期间，
	# 因为 用户线程 继续工作而导致 初始标记 产生变动的那一部分对象的标记记录。
	
# 重新标记 的 STW 会比 初始标记 STW 的时间稍长，
	# 但是会远远比 并发标记 阶段时间短。
	
# 主要使用 三色标记 里的 增量更新算法 做重新标记。
```

##### 并发清理

```shell
# 并发清理 是 GC 线程开始对 未标记的内存区域 做清理，
	# 并发清理 阶段中，用户线程 与 GC线程 并行工作。
	
	# 该阶段如果有 新增对象，将会被标记为 黑色，而不做任何 GC 处理。
```

##### 并发重置

```shell
# 重置本次 GC 过程中对 对象的标记数据。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596513157820.png)

#### CMS 优缺点

```shell
# CMS 优点:
	# 并发收集、减少 STW 时间、提高用户体验。
	
# CMS 缺点:
	# 并发阶段中，会和用户线程争抢 CPU 资源
	
	# 会产生浮动垃圾（在并发标记、并发清理 两个阶段，用户线程产生的新垃圾对象，只能留到下次 GC）
	
	# 使用 标记-清除 回收算法，收集结束会产生大量内存碎片
		# 但是 CMS 默认会在 标记-清除 之后，启动一次压缩整理
		# -XX:+UseCMSCompactAtFullCollection
		
	# GC 的不确定性
		# 可能在上一次 Full GC 还没执行完，下一次 Full GC 就被触发了，
		
		# 特别是在 并发标记 和 并发清除 这两个阶段 可能会出现，
		
		# 此时触发 concurrent mode failure,
		
		# 此时 GC线程 会 STW，启用 Serial Old 收集器进行 GC
```

#### CMS 核心参数

```shell
-XX:+UseConcMarkSweepGC # 启用 CMS

-XX:ConcGCThreads # 并发的 GC 线程数

-XX:+UseCMSCompactAtFullCollection # Full GC 之后做压缩整理

-XX:CMSFullGCsBeforeCompaction # 多少次 Full GC 之后压缩整理一次，默认是 0（每次都做）

-XX:CMSInitiatingOccupancyFraction # 老年代使用率达到 多少占比 后触发Full GC（默认92%）

-XX:+UseCMSInitiatingOccupancyOnly
	# 只使用设定的回收阈值（CMSInitiatingOccupancyFraction）
	# 如果不指定，JVM 仅在第一次 Full GC 使用设定值，后续则会自动调整
	
-XX:+CMSScavengeBeforeRemark
	# 在 CMS Full GC 之前，启动一次 Minor GC
	# 目的在于: 减少老年代对年轻代的引用，降低 CMS Full GC 的标记阶段时的开销，
	# 一般 CMS Full GC 耗时，80% 都在标记阶段
	
-XX:+CMSParallelInitialMarkEnabled # 在初始标记的时候，多线程执行，缩短 STW

-XX:+CMSParallelRemarkEnabled # 在重新标记的时候，多线程执行，缩短 STW
```

## JVM 参数设置示例

```shell
# 垃圾收集器组合使用: ParNew + CMS
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596520307025.png)

### 调优前的参数

```shell
# 对于 8G 内存服务器，一般分配 4G 给 JVM，首先 JVM 参数配置如下:
-Xms3072M -Xmx3072M -Xss1M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M      -XX:SurvivorRatio=8

# 这里的配置，正常来说没有什么问题，
	# 但是当电商系统，在做促销活动时，单位时间内进来大量请求线程，
	
	# 短时间内产生大量对象，Eden 区按 JVM 默认分配比例占整个堆内存的 1/3，
	
	# 那么 1G 的 Eden 区、8:1:1，Survivor 区就只有 100M，
	
	# 而对每秒产生最少 60MB 的系统来说，必然触发 对象动态年龄判断 机制，
	
	# 从而导致每几次 Minor GC 就会提前进入老年代一批对象，
	
	# 所以会频繁触发 Full GC，甚至可能导致 CMS concurrent mode failure 机制，
	
	# 直接 STW 所有用户线程，等待 Serial Old 收集器开启单线程 GC 收集所有垃圾对象。
	
# 结论:
	# 这里的配置，针对这里的应用场景，并不合适，需要我们调优来减少 Full GC 次数。
```

### 调优第一次后的参数

```shell
-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -XX:SurvivorRatio=8
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596520918095.png)

```shell
# 通过 JVM 参数调整，就基本避免了因 对象动态年龄判断 而导致提前进入老年代的问题。
```

### 调优第二次后的参数

```shell
# 大部分优化，都是让 短期存活 的对象，尽量留在 Survivor 区，而不要进入老年代，
	# 这样在 Minor GC 的时候，就让这些对象被回收，而不会进入到老年代、导致 Full GC.
```

```shell
# 对于 对象年龄 应该为多少才移到 老年代 比较合适？
	# 本例中，每次 minor gc 的间隔时间大概为 二三十秒，
	
	# 大多数对象一般在 几秒 内就会变成垃圾对象，
	
	# 所以完全可以将 默认的15岁 调小一点，
	
	# 比如设置成 5，基本上对象存活时间超过 1分钟、还没有被回收，
	
	# 完全可以认为这些对象是会存活时间较长的对象，
	
	# 提前一点移动到老年代，不要一直占用 年轻代的空间，损耗 复制 的性能。
```

```shell
# 对于 多大的对象 直接进入老年代？
	# 这个可以结合自己的业务系统来看，
	
	# 一般来说，设置为 1M 比较合适，
	
	# 很少有超过 1M 的大对象，
	
	# 一般都是 系统初始 的时候创建的 缓存对象，比如 List、Map 之类的。
```

```shell
-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 
-XX:PretenureSizeThreshold=1M
```

### 调优第三次后的参数

```shell
# JDK8 默认使用 Parallel + ParallelOld,
	# 如果内存较大（经验值是 超过 4个G），
	
	# 系统对 STW 停顿时间较为敏感，
	
	# 可以使用 ParNew + CMS。
```

```shell
# 对于老年代 CMS 的参数设置思考:
	# 当前系统有哪些对象可能会长期存活？
		# 一般就是 Spring 里的 Bean 容器、线程池对象、初始化缓存数据 等等...
		
		# 这些对象一般都在 100M 以内
		
	# 还有可能，在某次 Minor GC 完之后，还有超过 一两百M 的对象存活，
		# 那么就会触发 对象动态年龄判断机制，进入老年代一批对象
		
	# 还有可能，Minor GC 触发 老年代空间分配 担保机制，
		# 但一般来说，历次 Minor GC 后进入老年代的对象都不会太多，
		
		# 所以基本上不会在 Minor GC 触发之前，由于 老年代空间分配 担保失败而先产生一次 Full GC
		
	# 对于碎片整理:
		# 因为调优后 Full GC 产生时间间隔，基本都会大于 1个小时，
		
		# 所以几个小时才做一次 Full GC，完全可以每次都做碎片整理。
```

```shell
-Xms3072M -Xmx3072M -Xmn2048M -Xss1M -XX:MetaspaceSize=256M 
-XX:MaxMetaspaceSize=256M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=5 
-XX:PretenureSizeThreshold=1M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:CMSInitiatingOccupancyFraction=92 -XX:+UseCMSCompactAtFullCollection
-XX:CMSFullGCsBeforeCompaction=0
```

## 垃圾收集器底层标记算法实现

### 三色标记

```shell
# 在并发标记的过程中，GC线程 和 用户线程 同时工作，
	# 对象间的引用可能会发生变化，
	
	# 就可能产生 多标 和 漏标 的问题。
```

```shell
# 针对这种情况，JVM 引入了 "三色标记" 算法，
	# 把 GC Roots 可达性分析遍历对象过程中，
	
	# 按照 "是否访问过" 的条件标记成以下三种颜色。
		# 注意: 这里的颜色并不是说，JVM 就会在对象上标记一个颜色，
		# 而是在对象头指针中，用两个比特位来表示不同的状态，比如 00 01 10
```

```shell
# 黑色:
	# 表示 对象及子引用对象 已经被 GC Roots 引用链全部递归扫描过，
	
	# 表示该对象是 安全存活的，
	
	# 如果有 其他对象引用 指向了 黑色对象，无须再递归重新扫描一遍该对象，
	
	# 黑色对象不可能直接（不经过灰色对象）指向某个白色对象
```

```shell
# 灰色:
	# 表示 对象已经被 GC Roots 引用链访问过，
	
	# 但该对象 至少存在一个引用 还没有被扫描过。
```

```shell
# 白色:
	# 表示 对象尚未被 GC Roots 引用链访问过，即 GC Roots 不可达对象，
	
	# 在 可达性分析 刚开始的阶段，所有对象都是白色的，
	
	# 在 可达性分析 结束的时候，如果对象仍然是白色的，则代表垃圾对象。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596523679958.png)

```java
/**
 * 演示 GC 底层标记算法之三色标记
 * 不用在意代码编写规范细节。
 */
public class ThreeColorRemark {
  
  public static void main(String[] args) {
    A a = new A();
    
    // 开始做并发标记
    D d = a.b.d; // 1. 读
    a.b.d = null; // 2. 写
    a.d = d; // 3. 写
    
  }
  
  class A {
    B b = new B();
    D d = null;
  }
  
  class B {
    C c = new C();
    D d = new D();
  }
  
  class C {
    
  }
  
  class D {
    
  }
  
}
```

#### 多标 - 浮动垃圾

```shell
# 在 并发标记 过程中，GC线程 和 用户线程 同时工作，
	# 那么可能 某个用户线程 执行完毕，销毁线程栈，
	
	# 那 线程栈中的 局部变量表引用对象 在 初始标记 阶段已经被标记为 非垃圾对象（黑色或灰色）
	
	# 此时，线程栈销毁，局部变量表引用对象 可能已经变成了垃圾对象，
	
	# 但是本轮 GC 不会回收这部分对象内存，
	
	# 这部分 对象 被称为 浮动垃圾。
	
	# 浮动垃圾 并不会影响垃圾回收的正确性，
	
	# 只是需要等待下一轮 GC 才能被清除。
	
# 另外，在 并发标记 和 并发清理 两个阶段，用户线程产生的新对象，
	# 可能在 GC 完成之前就已经使用完毕，成为了垃圾对象，
	
	# JVM 通常的做法是将其全部当成黑色对象，
	
	# 本轮 GC 不会回收，
	
	# 这也算是 浮动垃圾 的一部分。
```

#### 漏标 - 读写屏障

```shell
# 在 GC 的并发过程中，
	# 初始标记 没有扫描到的对象、即白色对象，
	
	# 在做 并发标记 时，该白色对象被新的引用链引用了，
	
	# 如果此时，JVM 没有采取任何措施，那么就会造成 GC 掉 非垃圾对象，
	
	# 这是严重 Bug。
	
# JVM 解决 漏标 的两种方案:
```

##### 增量更新（Incremental Update）

```shell
# 当 黑色对象 插入新的指向 白色对象 的引用关系时，
	
# 就将这个 新插入的引用 记录下来，
	
# 等并发扫描结束之后、重新标记 STW 时，
	
# 再以这些 记录过的引用对象中 的 黑色对象 为根，重新扫描一次。
	
# 实际上，JVM 就是修改了该黑色对象的对象头指针，将其重新变为 灰色对象，等待下次扫描。
```

##### 原始快照（Snapshot At The Beginning 简写为: SATB）

```shell
# 当 灰色对象 要删除指向 白色对象 的引用关系时，

# 就将这个 要删除的引用 记录下来，

# 在并发扫描之后、重新标记 STW 时，

# 再以这些记录过的 引用关系 中的 灰色对象 为根，

# 重新扫描一次，这样就能扫描到 白色对象，

# 将 白色对象 直接标记为 黑色对象，
	# 目的就是让这种对象活过本轮 GC，不管是不是浮动垃圾，都交给下次 GC 处理。
```

```shell
# 以上无论是对 引用关系记录 的 插入还是删除，JVM 的记录操作都是通过 写屏障 实现的。
```

##### 写屏障

```shell
# 在给某个对象的成员变量赋值时，Hotspot 源码大概是这样:
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596524942679.png)

```shell
# 所谓的写屏障，其实就是指在 赋值操作前后，加入一些处理（参考AOP）
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596525001169.png)

##### 写屏障实现 SATB

```shell
# 当对象 B 的成员变量的引用发生变化时，
	# 比如 引用消失（a.b.d = null）,
	
	# JVM 会利用写屏障，将 B 原来成员变量的引用对象 D 记录下来
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596525109341.png)

##### 写屏障实现增量更新

```shell
# 当对象 A 的成员变量的引用发生变化时，
	# 比如 新增引用（a.d = b），
	
	# JVM 会利用写屏障，将 A 新的成员变量引用对象 D 记录下来
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596525208965.png)

##### 读屏障

```c++
oop oop_field_load(oop* field) {
  pre_load_barrier(field); // 读屏障 - 读取的前置操作
  return *field;
}
```

```shell
# 读屏障是直接针对第一步: D d = a.b.d，当读取成员变量时，一律记录下来:
```

```c++
void pre_load_barrier(oop* field) {
  oop old_value = *field;
  remark_set.add(old_value); // 记录读取到的旧对象
}
```

#### 总结

```shell
# 基于 GC Roots 可达性分析的垃圾回收器，
	# 几乎都借鉴了 三色标记 的算法思想，
	
	# 尽管实现的方式不尽相同，
		# 比如: 白色/黑色 集合一般都不会出现（但是有其他体现颜色的地方），
		
		# 灰色集合 可以通过 栈、队列、缓存日志 等方式进行实现，
		
		# 遍历方式可以是 广度遍历、深度遍历 等等...
```

```shell
# 对于 读写屏障，HotSpot 其 并发标记 时，对漏标的处理方案如下:
	# CMS: 写屏障 + 增量更新
	
	# G1: 写屏障 + SATB
	
	# ZGC: 读屏障
```

```shell
# 项目开发中，读写屏障 还有很多其他的功能，
	# 比如 写屏障 可以用于记录 跨代、区 引用的变化，
	
	# 读屏障 可以用于 支持移动对象、修改 其引用变量 原来的引用地址 两个操作的并发执行。
	
# 所以除了功能之外、还有性能的考虑、到底选择哪种方式，每款 GC 收集器都有自己的想法。
```

#### 为什么 G1 用SATB、CMS 用增量更新？

```shell
# 个人理解:
	# SATB 相对 增量更新 来说、效率更高（但是可能造成更多的浮动垃圾）
		# 因为不需要在 重新标记阶段 再次深度扫描 被删除引用 的对象。
		
	# G1 因为很多对象都位于不同的 region，
	
	# CMS 就一块老年代内存空间，
	
	# 如果用 增量更新 在 重新标记 的阶段再次做深度扫描的话，
		# 对 G1 来说，会存在更多 跨代引用 的问题，
		
		# 所以 G1 选择 SATB，直接抛出 浮动垃圾，让下次 GC 处理。
```

### 记忆集与卡表

```shell
# 在 Minor GC 时，可达性分析 扫描过程中，可能会碰到跨代引用的对象，
	# 即 老年代对象 引用 年轻代对象，
	
	# 这种情况如果又去老年代做 GC Roots 引用链扫描的话，效率就太低了。
	
	# 所以，JVM 在 年轻代 引入了 记忆集（Remember Set）的数据结构，
	
	# 专门用于记录 非收集区（Old） 到 收集区（Young） 的指针集合，
	
	# 避免把整个 Old 区加入 GC Roots 扫描范围。
	
	# 事实上，并不只有 Young、Old 之间才有 跨代引用的 问题，
	
	# 所有涉及 部分区域 收集行为的 GC 收集器，都会面临相同的问题。
```

```shell
# GC 收集过程中，
	# 收集器 只需要通过 记忆集 判断出 某一块非收集区域 是否存在指向 收集区域 的指针即可，
	
	# 无需了解 跨代引用 指针的全部细节。
```

```shell
# HotSpot 使用 卡表（CardTable）的方式实现 记忆集，
	# 也是目前最常用的一种方式。
	
	# 关于 卡表 与 记忆集 的关系，可以类比 HashMap 和 Map 的关系。
	
# 卡表 是使用了一个 字节数组（ CARD_TABLE[] ） 实现，
	# 每个 元素 对应着 其标识的内存区域 一块特定大小的内存块，这称之为 卡页。
	
	# HotSpot 使用的 卡页 是 2^9 大小，即 512 字节。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596527001382.png)

```shell
# 一个 卡页 中，可包含多个对象，
	# 只要有一个 对象的字段 存在 跨代指针，
	
	# 其对应的 卡表的元素标识 就会变成 1，标识该 元素变脏， 否则为 0，
	
	# GC 时，只需要 筛选本收集区里、卡表中变脏的元素 加入 GC Roots 集合即可。
```

```shell
# 卡表的维护:
	# 卡表变脏 的意义如上所述，
	
	# JVM 是通过 写屏障 来维护卡表的状态，
	
	# 即发生 引用字段复制 时，更新 卡表对应的标识 为1。
```

