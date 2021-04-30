[TOC]

# 周阳 - Java 底层<下>

## JVM + GC

### 复习

#### JVM 体系结构概览

![UTOOLS1571120689961.png](https://i.loli.net/2019/10/15/ZI6gcWXGMqVHOnu.png)

```shell
# 类加载器
	# 种类
		# 启动类加载器
		# 扩展类加载器
		# 应用类加载器
		# 自定义加载器
		
	# 双亲委派机制
		# 能让父加载器加载的就让父加载器加载
		# 沙箱安全机制<起个包名叫 java.lang，起个类叫 Object，写个 main 方法看看？>
```

### JVM 垃圾回收的时候如何确定垃圾？是否知道什么是 GC Roots？

```shell
# 什么是垃圾？
	# 内存中已经不再被使用到的空间就是垃圾
	
# 要进行垃圾回收，如何判断一个对象是否可以被回收？
	# 引用计数法
		# 容易产生循环引用的问题，弃用
		
	# 可达性分析
		# 给定一个集合(GC Roots)的引用作为根出发，通过引用关系遍历对象图
			# 能被遍历到的(可到达的) 对象就被判断为存活
			# 没有被遍历到的就被判定为死亡
			
		# GC Roots
			# 一组必须活跃的引用
			# 虚拟机栈中的引用对象
			# 方法区中的类静态属性引用的对象
			# 方法区中常量引用的对象
			# 本地方法栈中引用的对象
```

### 如何盘点查看 JVM 系统默认值

#### JVM 的参数类型

```shell
# 标配参数
	-version
	-help
	-showversion
	
# X 参数(了解即可)
	-Xint : 解释执行
	-Xcomp : 编译执行
	-Xmixed : 混合模式

# XX 参数(重点)
	# 布尔类型
		# -XX: + 或者 - 某个属性值
			# + 表示开启
			# - 表示关闭
	
	# KV 设值类型
		# -XX:key=value
		
	# 坑题
		# -Xms 和 -Xmx 是简写
		# -Xms = -XX:InitialHeapSize
		# -Xmx = -XX:MaxHeapSize
```

### Java 常用命令

```shell
# jps -l
	# 列出系统中 Java 进程
	
# jinfo
	# 查看 Java 进程信息
	# 例: jinfo -flag PrintGCDetails [Java 进程号]
		# 查看该 Java 进程是否开启打印 GC 信息
	# 例: jinfo -flag UseSerialGC [Java 进程号]
		# 查看该 Java 进程是否使用串行垃圾回收器
	# 例: jinfo -flag MetaspaceSize [Java 进程号]
		# 查看该 Java 进程的元空间大小<默认元空间大小为21M 内存>
	# 例: jinfo -flag MaxTenuringThreshold [Java 进程号]
		# 查看该 Java 进程年轻代需要经历多少次轻GC 才能进入老年代<默认15 次>
	# jinfo -flags [Java 进程号]
		# 查看当前 Java 进程所有参数配置
		
# java -XX:+PrintFlagsInitial
	# 查看 JVM 所有初始化参数
	
# java -XX:+PrintFlagsFinal
	# 查看 JVM 当前参数配置
	
# java +XX:+PrintCommandLineFlags
	# 查看 JVM 常用命令行参数
	# 主要查看 JVM 默认的垃圾回收器
	# jdk8 默认垃圾回收器是 ParallelGC<并行垃圾回收器>
	
# 注意: 值有两种表达方式
	# = 是没有被修改过的
	# := 人为修改或 JVM修改过的值
```

### Java 查看JVM 参数以及修改参数运行

```shell
# java -XX:+PrintFlagsFinal -XX:MetaspaceSize=512m Test
	# java 运行 Test 程序并打印系统参数，修改元空间的大小为 512m
```

### 谈谈你工作中用过的 JVM 常用基本配置参数有哪些？

#### 堆内存初始大小快速复习

```shell
# JDK8 之后，JVM 堆空间中的永久代被元空间取代
	
# 元空间<JDK8>与永久代<JDK7>最大的区别在于
	# 永久代使用 JVM 堆内存
	# 元空间并不在 JVM 中，而是使用本机物理内存
	
# 默认情况下，元空间的大小仅受本地内存限制
	# 类的元数据放入本地内存，常量池和类的静态变量放入 JVM 堆中
	# 这样可以加载多少类的元数据就不再由 MaxPermSize 控制
	# 而是系统的实际可用空间来控制
```

![UTOOLS1572228209131.png](https://i.loli.net/2019/10/28/VdFbqABgHOS4wWv.png)

#### JVM 常用基本配置参数

```shell
# -Xms
	# 堆空间初始大小内存，默认为物理内存的 1/64
	# 等价于 -XX:InitialHeapSize
	# 一般来说，最好设置和 -Xmx 一样的值
	
# -Xmx
	# 堆空间最大分配内存，默认为物理内存 1/4
	# 等价于 -XX:MaxHeapSize
	# 一般来说，最好设置和 -Xms 一样的值
	
# -Xss
	# 设置单个线程栈的大小，一般默认为 512k - 1024k
	# 等价于 -XX:ThreadStackSize
	# 具体大小决定于底层操作系统平台<基本上都是 1024k = 1m>
	
# -Xmn
	# 设置堆内存中年轻代内存空间的大小
	# 一般来说使用默认值即可，年轻代占堆内存空间的1/3，老年代占2/3
	
# -XX:MetaspaceSize
	# 设置元空间大小
	# 元空间的本质和永久代类似，都是对 JVM 规范中方法区的实现
	# 一般来说，最好将该参数使用内存大小调大，默认初始化大小为 21m
	
# -XX:+PrintGCDetails
	# 输出详细 GC 收集日志信息
	# GC
	# FullGC 
	
# -XX:SurvivorRatio
	# 设置新生代中 eden 和 S0/S1 空间的比例
	# 默认 -XX:SurvivorRatio=8    <Eden:S0:S1 =8:1:1>
	# SurvivorRatio 值就是设置 eden 区的比例占多少，S0/S1 相同
	
# -XX:NewRatio
	# 配置年轻代与老年代在堆结构的占比
	# 默认 -XX:NewRatio=2
	# 新生代1/3，老年代2/3
	# NewRatio 值就是设置老年代的占比，剩下的1给新生代
	
# -XX:MaxTenuringThreshold
	# 设置垃圾最大年龄<S0 -> S1 如此反复 xx 次进入老年代>
	# 默认为 15
	# 必须设置为 0 - 15 之间
```

![UTOOLS1572230295909.png](https://i.loli.net/2019/10/28/a1fh4q3CL58Irzo.png)

#### 典型设置案例

```shell
# 最常用的 JVM 参数配置命令演示
-Xms128m -Xmx4096m -Xss1024k -XX:MetasapceSize=512m -XX:+PrintCommandLineFlags -XX:+PrintGCDetails -XX:+UseSerialGC
	
	# 堆内存初始化大小 128m、最大 4096m，单个线程栈大小 1024k，元空间 512m，打印参数、GC 详情、使用串行化 GC 处理器
```

### 强引用、软引用、弱引用、虚引用分别是什么？

```shell
# 整体架构
```

![UTOOLS1572233748892.png](https://i.loli.net/2019/10/28/4Od5X82VM6izvtY.png)

```shell
# 强引用<默认支持模式>
	# 当内存不足，JVM 开始垃圾回收，对于强引用的对象，就算是出现了 OOM 也不会对该对象进行回收
	
	# 强引用是最常见的普通对象引用，只要还有强引用指向一个对象，就表明该对象还“活着”，GC 就不会碰这种对象。Java 中最常见的就是强引用，把一个对象赋给一个引用变量，这个引用变量就是一个强引用。当一个对象被强引用变量引用时，就处于可达状态，不能被GC，即时该对象以后永远都不会被用到 JVM 也不会回收。因此强引用是造成 Java 内存泄露的主要原因之一。
	
	# 对于一个普通的对象，如果没有其他引用关系，只要超过了引用的作用域或者显示地将相应引用赋值为null，一般认为就是可以被GC 了<具体回收时机还是要看垃圾收集策略>
	
# 软引用
	# 软引用是一种相对强引用弱化了一些的引用，需要用 java.lang.ref.SoftReference 类来实现
	# 可以让对象豁免一些垃圾收集
	# 对于只有软引用的对象来说
		# 系统内存充足时，它不会被回收
		# 系统内存不足时，它会被回收
	# 软引用通常用在对内存敏感的程序中，比如高速缓存就有用到软引用，内存够用就保留，不够就回收
	
# 弱引用
	# 弱引用需要用 java.lang.ref.WeakReference 类来实现，它比软引用的生存期更短
	# 对于只有弱引用的对象来说，只要垃圾回收机制一运行，不管 JVM 内存是否足够，都会回收该对象
	
# 软应用和弱引用的适用场景
	# 应用需要读取大量的本地图片
	# 用一个 HashMap 来保存图片的路径和相应图片对象关联的软引用之间的映射关系，在内存不足时，JVM 会自动回收这些缓存图片对象所占用的空间，从而有效地避免了 OOM 的问题
	
# 你知道弱引用的话，能谈谈 WeakHashMap 吗？
	# WeakHashMap 中内部类 Entry<K,V> 是继承了 WeakReference 弱引用的
	# 当 GC 后，WeakHashMap 内 Entry 对象就会被 GC 回收
	
# 虚引用
	# 虚引用需要 java.lang.ref.PhantomReference 类来实现
	# 与其他几种引用不同，虚引用并不会决定对象的生命周期
	# 如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收
	# 它不能单独使用也不能通过它访问对象，虚引用必须和引用队列(ReferenceQueue) 联合使用
	
	# 虚引用的主要作用是跟踪对象被垃圾回收的状态
	# 仅仅提供一种确保对象被 finalize 以后，做某些事情的机制
	# 设置虚引用关联的唯一目的，就是在该对象被 GC 的时候收到一个系统通知或者后续添加进一步的处理
	# Java 允许使用 finalize() 方法在 GC 之前做必要的清理工作
	
# ReferenceQueue 引用队列
	# 对象在被回收之前，需要被引用队列保存下
```

### 请你谈谈对 OOM 的认识

```shell
# java.lang.StackOverflowError
	# 栈溢出
	# 演示方法 : 递归调用方法本身，线程栈默认为 1024k,无限存储栈帧导致栈溢出

# java.lang.OutOfMemoryError:Java heap space
	# 堆内存溢出
	# 演示方法 : byte[] bytes = new byte[1024 * 1024 * 1024 * 1024];

# java.lang.OutOfMemoryError:GC overhead limit exceeded
	# GC 回收时间过长时，会抛出 OutOfMemoryError
	# 过长的定义是，超过 98% 的时间用来做 GC 且回收了不到 2% 的堆内存
	# 连续多次 GC 都只回收了不到 2% 的极端情况下才会抛出 GC overhead limit
	# 就是 GC 清理的这点内存很快就会被占满，迫使 GC 再次执行，这样就形成恶性循环
	# CPU 使用率一直是 100%，而 GC 却没有任何成果

# java.lang.OutOfMemoryError:Direct buffer memeory
	# 写 NIO 程序经常使用 ByteBuffer 来读取或者写入数据
	# 这是一种基于通道(Channel) 与缓冲区(Buffer) 的I/O 方式
	# 它可以使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆里面的 DirectByteBuffer 对象作为这块内存的引用进行操作
	# ByteBuffer.allocate(capability) 第一种方式是分配 JVM 堆内存，属于 GC 管辖范围，由于需要拷贝所以速度相对较慢
	# ByteBuffer.allocateDirect(capability) 第二种方式是分配 OS 本地内存，不属于 GC 管辖范围，由于不需要内存拷贝所以速度相对较快
	# 但如果不断分配本地内存，堆内存很少使用，那么 JVM 就不需要执行 GC，DirectByteBuffer 对象们就不会被回收，此时堆内存充足，但本地内存可能已经使用光了，再次尝试分配本地内存就会出现OutOfMemoryError
	
	# 演示方法 : ByteBuffer buffer = ByteBuffer.allocateDirect(6 * 1024 * 1024)

# java.lang.OutOfMemoryError:unable to create new native thread
	# 高并发请求服务器时，经常出现该异常
	# 准确的讲，native thread 异常与对应的平台有关
	# 导致原因
		# 1. 应用创建了太多线程了，一个应用进程创建多个线程，超过系统承载极限
		# 2. 服务器并不允许你的应用程序创建这么多线程，Linux 系统默认允许单个进程可以创建的线程数是1024个，Java 应用创建线程数超过这，就会报该错误
		
	# 解决办法
		# 1. 降低应用程序创建线程数量，分析应用是否真的需要创建这么多线程，如果不是，降到最低
		# 2. 对应有的应用，确实需要创建很多线程，远超过 Linux 系统默认的1024个线程的限制，此时可以通过修改 Linux 服务器配置，扩大 Linux 默认限制

# java.lang.OutOfMemoryError:Metaspace
	# JDK8 Metaspace 取代老年代，使用本地内存
	# 存放如下信息
		# JVM 加载的类信息
		# 常量池
		# 静态变量
		# 即时编译后的代码
	# 演示方法 : Enhancer 利用 cglib 动态字节码技术不断反射调用目标类
```

### 垃圾收集器

#### GC 垃圾回收算法和垃圾收集器的关系？分别是什么请你谈谈

```shell
# GC 算法(引用技术/复制/标清/标整) 是内存回收的方法论，垃圾收集器就是算法落地实现

# 因为目前为止还没有完美的收集器出现，只是针对具体应用最合适的收集器，进行分带收集

# 4种主要垃圾收集器
	# Serial 串行垃圾回收器
		# 为单线程环境设计且只使用一个线程进行垃圾回收，会暂停所有的用户线程，不适合服务器环境
	
	# Parallel 并行垃圾回收器<JDK 8 默认>
		# 多个垃圾收集线程并行工作，此时用户线程是暂停的，适用于科学计算|大数据|首台处理 等弱交互场景
	
	# CMS 并发标记清除垃圾回收器
		# 用户线程和垃圾收集线程同时执行(不一定是并行，可能交替执行)，不需要停顿用户线程
		# 互联网公司多用它，适用对响应时间有要求的场景
	
	# G1
		# G1 将堆内存分割成不同的区域然后并发的对其进行垃圾回收
```

### 怎么查看服务器默认的垃圾收集器是哪个？

### 生产上如何配置垃圾收集器的？

### 谈谈你对垃圾收集器的理解？

```shell
# 四大垃圾回收算法思想
	# 引用计数
	# 复制拷贝
	# 标记清除
	# 标记整理
	
# 落地实现
	# 串行回收
	# 并行回收
	# 并发回收
	# G1
```

```shell
# 怎样查看默认的垃圾回收器？
	# java -xx:+PrintCommandLineFlags 
```

```shell
# 默认的垃圾收集器有哪些？
	# UseSerialGC
	# UseSerialOldGC
	# UseParallelGC
	# UseConcMarkSweepGC
	# UseParNewGC
	# UseParallelOldGC
	# UseG1GC
```

![UTOOLS1572314245909.png](https://i.loli.net/2019/10/29/XTWBCDUPfApn2wj.png)

### 新生代使用的 GC

```shell
# 串行GC <Serial | Serial Copying>
	# 一个单线程的收集器，在进行垃圾收集的时候，必须暂停其他所有的工作线程直到它收集结束
	# 现在基本不用了，略

# 并行GC <ParNew>
	# 使用多线程进行垃圾回收，在垃圾收集时，会暂停其他所有的工作线程直到它收集结束
	# ParNew 其实就是 Serial 收集器新生代的并行多线程版本
	# 最常见的应用场景是配合 老年代的 CMS GC 工作，其余的行为和 Serial 完全一样
	# 是大部分 JVM 运行在 Server 模式下新生代的默认垃圾收集器
	# 常用对应 JVM 参数 : -XX:+UseParNewGC
		# 启用 ParNew 收集器，只影响新生代的收集，不影响老年代
		# 开启上述参数后，会使用 ParNew(Young 区用) + Serial Old 的收集器组合<不推荐>
		# 新生代使用复制算法，老年代采用标记-整理算法

# 并行回收 <Parallel | Parallel Scavenge>
	# Parallel Scavenge 收集器类似 PreNew 也是一个 新生代 垃圾收集器，使用复制算范
	# 也是一个并行的多线程的垃圾收集器，俗称吞吐量优先收集器
	# 一句话 : 串行收集器在新生代和老年代的并行化
	# 重点关注 :
		# 可控制的吞吐量<Thoughput = 运行用户代码时间/(运行用户代码时间 + 垃圾收集时间)>，也即比如程序运行 100 分钟，垃圾收集时间 1 分钟，吞吐量就是 99%)。
		# 高吞吐量意味着高效利用 CPU 的时间，它多用于在后台运算而不需要太多交互的任务
		
		# 自适应调节策略也是 ParallelScavenge 收集器与 ParNew 收集器的一个重要区别。<自适应调节策略: JVM 会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间(-XX:MaxGCPauseMillis) 或最大的吞吐量>
	
	# 常用 JVM 参数: -XX:+UseParallelGC 或 -XX:+UseParallelOldGC<可互相激活>，使用 Parallel Scanvenge 收集器
	# 开启该参数后，新生代使用复制算法，老年代使用标记 - 整理算法
```

### 老年代使用的GC

```shell
# 串行 GC <Serial Old | Serial MSC>

# 并行 GC <Parallel Old | Parallel MSC>
	# Parallel Old 收集器是 Parallel Scavenge 的老年代版本
	# 使用多线程的 标记-整理 算法，Parallel Old 收集器在 JDK1.6 才开始提供
	# JDK8 默认使用的垃圾收集器

# 并发标记清除 GC <CMS>
	# 是一种以获取最短回收停顿时间为目标的收集器
	# 适合应用在互联网站或 B/S 系统的服务器上，这类应用尤其重视服务器的响应速度，希望系统停顿时间最短
	# CMS 非常适合堆内存大、CPU 核数多的服务器端应用，也是 G1 出现以前大型应用的首选收集器
	# 开启该收集器的 JVM 参数: -XX:+UseConcMarkSweepGC
		# 开启该参数后会自动将 -XX:+UseParNewGC 打开
		# 使用 ParNew(Young 区用) + CMS(Old 区用) + Serial Old 的收集器组合
			# Serial Old 将作为CMS 出错的后备收集器
			
	# 四步过程
		# 初始标记<CMS initial mark>
			# 只是标记一下 GC Roots 能直接关联的对象，速度很快，仍然需要暂停所有的工作线程
			
		# 并发标记<CMS concurrent mark，和用户线程一起>
			# 进行 GC Roots 跟踪的过程，和用户线程一起工作，不需要暂停工作线程。主要标记过程，标记全部对象。	
			
		# 重新标记<CMS remark>
			# 为了修正在并发标记期间，因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，仍然需要暂停所有的工作线程
			# 由于并发标记时，用户线程依然运行，因此在正式清理前，再做修正。
			
		# 并发清除<CMS concurrent sweep，和用户线程一起> 
			# 清除 GC Roots 不可达对象，和用户线程一起工作，不需要暂停工作线程。基于标记结果，直接清理对象
			# 由于耗时最长的并发标记和并发清除过程中，垃圾收集线程可以和用户现在一起并发工作，所以总体上来看 CMS 收集器的内存回收和用户线程是一起并发地执行。
			
	# 优缺点 :
		# 优点 : 并发收集低停顿
		# 缺点 : 并发执行，对CPU资源压力大，采用的标记清除算法会导致大量碎片
		
	# 由于并发执行，CMS 在收集与应用线程会同时增加对堆内存的占用，也就是说，CMS 必须要在老年代堆内存用尽之前完成垃圾回收，否则 CMS 回收失败时，将触发担保机制，串行老年代收集器将会以 SWT 的方式进行一次 GC，从而造成较大停顿时间
	
	# 标记清除算法无法整理空间碎片，老年代空间会随着应用时长被逐步耗尽，最后将不得不通过担保机制对堆内存进行压缩。 CMS 也提供了参数  -XX:CMSFullGCCsBeForeCompaction(默认0，即每次都进行内存整理) 来指定多少次 CMS 收集之后，进行一次压缩的 Full GC
```

### GC 之如何选择垃圾收集器

```shell
# 单 CPU 或小内存，单机程序
	# -XX:+UseSerialGC <不推荐>
	
# 多 CPU 需要最大吞吐量，如后台计算型应用
	# -XX:+UsePrallelGC 
	# -XX:+UseParallelOldGC
	
# 多 CPU 追求低停顿时间，需快速响应如互联网应用
	# -XX:+UseConcMarkSweepGC
	# -XX:+ParNewGC
```

![UTOOLS1572333288981.png](https://i.loli.net/2019/10/29/8S6Da1Uo9R4qN3G.png)

### GC 之G1收集器

```shell
# 以前收集器特点
	# 年轻代和老年代是各自独立且连续的内存块
	# 年轻代收集使用单 eden+S0+S1 进行复制算法
	# 老年代收集必须扫描整个老年代区域
	# 都是以尽可能少而快速的执行 GC 为设计原则
```

#### G1 是什么

```shell
# 像 CMS 收集器一样，能与应用程序线程并发执行
	# 整理空闲空间更快
	# 需要更多的时间来预测 GC 停顿时间
	# 不希望牺牲大量的吞吐性能
	# 不需要更大的 Java Heap
	
# G1 收集器的设计目标是取代 CMS 收集器，它同 CMS 相比，在以下方便更加出色
	# G1 是一个有整理内存过程的垃圾收集器，不会产生很多内存碎片
	# G1 是 Stop The World 更可控，在停顿时间上添加了预测机制，用户可以指定期望停顿时间
	
# JDK9 默认的 GC 处理器
	# 主要改变是 Eden,Survivor 和 Tenured 等内存区域不再是连续的了，而是变成了一个个大小一样的 region，每个 region 从 1M 到 32 不等，一个 region 有可能属于 Eden,Survivor 或者 Tenured 内存区域
```

#### G1 特点

```shell
# G1 能充分利用多 CPU、多核环境硬件优势，尽量缩短 STW

# G1 整体上采用标记-整理算法，局部是通过复制算法，不会产生内存碎片

# 宏观上看 G1 之中不再区分年轻代和老年代，把内存划分成多个独立的子区域<Region>

# G1 里讲整个的内存区都混合在一起了，但其本身依然在小范围内要进行年轻代和老年代的区分，保留了新生代和老年代，但它们不再是物理隔离的，而是一部分 Region 的集合且不需要 Region 是连续的，也就是说依然会采用不同的 GC 方式来处理不同的区域

# G1 虽然也是分代收集器，但整个内存分区不存在物理上的年轻代与老年代的区别，也不需要完全独立的 survivor 堆做复制准备，G1 只有逻辑上的分代概念，或者说每个分区都可能随 G1 的运行在不同代之间前后切换
```

#### G1 底层原理

```shell
# Region 区域化底层收集器
	# 化整为零，避免全内存扫描，只需要按照区域来进行扫描即可
	
	# 核心思想是将整个堆内存区域分成大小相同的子区域(Region)，在 JVM 启动时会自动设置这些子区域的大小，在堆的使用上,G1 并不要求对象的存储一定是物理上连续的只要逻辑上连续即可，每个分区也不会固定地为某个代服务，可以按需在年轻代和老年代之间切换。启动时可以通过参数 -XX:G1HeapRegionSize=n 可指定分区大小(1MB   - 32MB,且必须是 2 的幂)，默认且最多将整堆划分为 2048 个分区，最大支持内存为 32MB * 2048 = 64G 内存
	
	# 这些 Region 的一部分包含新生代，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到老年代或者 Survivor 空间
	
	# 这些 Region 的一部分包含老年代，G1 收集器通过将对象从一个区域复制到另外一个区域，完成了清理工作，这就意味着，在正常的处理过程中，G1 完成了堆的压缩<至少是部分堆的压缩>，这样就不会有 CMS 内存碎片问题的存在了
	
	# 在 G1 中，还有一种特殊的区域，叫 Humongous<巨大的> 区域，如果一个对象占用的空间超出了分区容量 50% 以上，G1 就会人为该对象是一个巨型对象。这些巨型对象默认直接会被分配在老年代，但是如果它是一个短期存在的巨型对象，就会对 GC 造成负面影响。为了解决这个问题，G1 划分了一个 Humongous 区，它用来专门存放巨型对象。如果一个 H 区装不下该巨型对象，那么 G1 会寻找连续的 H 分区来存储。为了能找到连续的 H 区，有时候不得不启动 Full GC
	
# 回收步骤
	# G1 收集器下的 Young GC
		# 针对 Eden 区进行收集，Eden 区耗尽后会被触发，主要是小区域收集 + 形成连续的内存块，避免内存碎片
		# Eden 区的数据移动到 Survivor 区，加入出现 Survivor 区空间不够，Eden 区数据全部会晋升到 Old 区
		# Survivor 区的数据移动到新的 Survivor 区，数据晋升到 Old 区
		
		# 最后 Eden 区收拾干净了，GC 结束，用户的应用程序继续执行
	
# 四步过程
	# 初始标记: 只标记 GC Roots 能直接关联到的对象
	# 并发标记: 进行 GC Roots Tracing 的过程
	# 最终标记: 修正并发标记期间，因程序运行导致标记发生变化的那一部分对象
	# 筛选回收: 根据时间来进行价值最大化的回收
```

![UTOOLS1572335586734.png](https://user-gold-cdn.xitu.io/2019/10/29/16e168111d07a637?w=430&h=250&f=png&s=64147)

#### G1 常用配置参数

```shell
# -XX:+UseG1GC

# -XX:G1HeapRegionSize=n:
	# 设置的G1 区域的大小。值是2的幂，范围是 1MB 到 32MB
	# 目标是根据最小的 Java 堆大小划分出约 2048 个区域
	
# -XX:MaxGCPauseMillis=n:
	# 最大 GC 停顿时间，这是个软目标，JVM 将尽可能<但不保证>停顿小于这个时间
	
# -XX:InitiatingHeapOccupancyPercent=n:
	# 堆占用了多少的时候就触发 GC，默认为 45
	
# -XX:ConGCThreads=n:
	# 并发 GC 使用的线程数
	
# -XX:G1ReservePercent=n:
	# 设置作为空闲时间的预留内存百分比，以降低目标空间溢出的风险，默认值是10%
	
# 一般生产上这样配置就行
	# -XX:+UseG1GC  -Xmx32g   -XX:MaxGCPauseMillis=100
```

#### G1 和 CMS的比较

```shell
# 比起 CMS 有两个优势
	# G1 不会产生内存碎片
	# 是可以精确控制停顿，该收集器是把整个堆(新生代、老年代) 划分成多个固定大小的区域，每次根据允许停顿的时间去收集垃圾最多的区域
```

### JVM GC 结合 SpringBoot 微服务优化

```shell
# 公式
	java -server [JVM 的各种参数] -jar [jar Or war 包名]
```

### 生产环境服务器变慢，诊断思路和性能评估谈谈？

```shell
# Linux 命令
	# 整机 : top
	# CPU : vmstat
	# 内存 : free
	# 硬盘 : df
	# 磁盘IO : iostat
	# 网络IO : ifstat
	
# 结合 Linux 和 JDK 命令一块分析
	# 1. 先用 top 命令分析占用 CPU 和 Memory 最高的进程
	
	# 2. jps 进一步定位，得知具体的后台Java程序
	
	# 3. 定位到具体线程或者代码
		# ps -mp [进程] -o THREAD,tid,time
			# -m 显示所有的线程
			# -p pid 进程使用 cpu 的时间
			# -o 该参数后使用户自定义格式
			
	# 4. 将需要的 线程ID 转换为 16进制格式<英文小写格式>
	
	# 5. jstack 进程ID | grep tid -A60
```

