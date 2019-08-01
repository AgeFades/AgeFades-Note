# 慕课 - Java 生产环境下性能监控与调优详解

## 基于 JDK 命令行工具的监控

### 	JVM 的参数类型

#### 		标准参数（各个 JDK 版本稳定参数）

```shell
-help

-server -client

-version -showversion

-cp -classpath
```

#### 		X 参数

```shell
-Xint # 解释执行	
-Xcomp # 第一次使用就编译成本地代码
-Xmixed # 混合模式，JVM 自己决定是否编译成本地代码 默认
```

#### 		XX 参数

```shell
# 主要用于 JVM 调优和 Debug

# Boolean 类型
# 格式 -XX:[+-]<name> 表示启用或禁用 name 属性 例
-XX:+UseConcMarkSweepGC # 启用 CMS 垃圾处理器
-XX:+UseG1GC # 启用 G1 垃圾处理器

# 非Boolean 类型
# 格式 -XX:<name>=<value> Key Value 类型 例
-XX:MaxGCPauseMillis=500 # GC 最大的暂停时间为 500ms
-XX:GCTimeRatio=19

# -Xmx -Xms
-Xmx # 等价于 -XX:MaxHeapSize ，设置最大堆内存
-Xms # 等价于 -XX:InittailHeapSize ，设置最小堆内存
```

### 	运行时 JVM 参数查看

```shell
-XX:+PrintFlagsInitial # 查看初始值
-XX:+PrintFlagsFinal # 查看最终值
-XX:+UnlockExperimentalVMOptions # 解锁实验参数
-XX:+UnlockDiagnosticVMOptions # 解锁诊断参数
-XX:+PrintCommandLineFlags # 打印命令行参数

jps # 展示 Linux 中运行的 Java 进程
jinfo # 查看运行时的 Java 进程参数
jinfo -flag MaxHeapSize xxx # 检测 Java 进程 pid 的最大堆内存
jinfo -flags xxx # 检查 Java 进程 pid 的参数信息
```

#### 		PrintFlagsFinal : ​		![UTOOLS1564626179767.png](https://i.loli.net/2019/08/01/5d424d05ba9af45109.png)

### 	jstat 查看虚拟机统计信息

```shell
jstat -class # 查看类加载信息
jstat -compiler # 查看编译信息
jstat -gc # 查看 GC 信息
```

### 	jmap + MAT 实战内存溢出

```shell
OutOfMemory: Heap # 堆内存溢出
OutOfMemory: MetaSpace # 元空间内存溢出

# 内存溢出时，如何导出内存映像文件
# 内存溢出自动导出
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=./

# 使用 jmap 命令手动导出
jmap -dump:format=b,file=heap.hprof xxx[进程 pid]
jmap -heap xxx # 查看Java进程的堆空间使用情况

# MAT 分析内存溢出
下载软件，导入内存文件，工具分析
```

### 	jstack 实战死循环与死锁

```shell
jstack xxx > xxx.txt # 将Java 进程 pid 线程状态打印成一个文件
# 通过 Linux top 命令查看 Java 进程中各个线程的资源使用率，将10进制的 pid 转换成 16进制
# 去 jstack 中查询对应的代码片段
```

## 基于 JVisualVM 的可视化监控

```shell
JRE 自带可视化监控工具
可实现 jmap、jstack 等命令操作并提供可视化分析
监控热点方法、各项资源的利用情况
```

## 基于 Btrace 的监控调试

```shell
BTrace 可以动态向目标应用程序的字节码注入追踪代码
```

## JVM 层 GC 调优

### 	JVM 的内存结构

#### 		运行时数据区

![UTOOLS1564639068899.png](https://i.loli.net/2019/08/01/5d427f5ecdd6889110.png)

#### 		程序计数器 PC Register

​			JVM 支持多线程同时执行，每一个线程都有自己的 PC Register

​			线程正在执行的方法叫做当前方法，PC Register 存放的就是当前正在执行的指令地址

#### 		虚拟机栈 JVM Stacks

​			线程私有，生命周期与线程相同

​			栈描述的是 Java 方法执行的内存模型

​			每个方法在执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口

​			每个方法从调用直至执行完成，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程

#### 		本地方法栈 Native Method Stacks

​			与 JVM Stacks 类似

​			不同的是 Native 执行的是本地方法，需要调用操作系统的方法

#### 		堆 Heap

​			JVM 管理中内存最大的一块，被所有线程共享，JVM 启动时创建

​			堆的唯一目的就是存放对象实例，可以处于物理上不连续的内存，只要逻辑上是连续的即可

![UTOOLS1564639797164.png](https://i.loli.net/2019/08/01/5d428236a635392145.png)

#### 		元空间 MetaSpace

​			保存 Class、Package、Method、Field、字节码、常量池、符号引用等

​			CSS : 32 位指针的 Class

​			CodeCache : JIT 编译后的本地代码、JNI 使用的 C代码

#### 		方法区 Method Area

​			也是各个线程共享的内存区域

​			存储已被JVM 加载的类信息、常量、静态变量、即时编译器编译后的代码等数据

​			JVM 将其描述成 Heap 的一个逻辑部分

#### 		常量池 RunTime Constant Pool

​			方法区的一部分，Class 文件中除了有类的版本、字段、方法、接口等描述信息外

​			还有一项常量池，用于存放编译器生成的各种字面量和符号引用

### 	常用参数

```shell
-Xms -Xmx
-XX:NewSize -XX:MaxNewSize
-XX:NewRatio -XX:SurvivorRatio
-XX:MetaspaceSize -XX:MaxMetaspaceSize
-XX:+UseCompressedClassPointers
-XX:CompressedClassSpaceSize
-XX:InitialCodeCacheSize
-XX:ReservedCodeCacheSize
```

### ​	垃圾回收算法

​		**思想** : 

​			枚举根节点，做可达性分析

​		**根节点** :

​			类加载器、Thread、虚拟机栈的本地变量表、static 成员、常量引用、本地方法栈的变量等

#### 		标记清除

##### 			算法

​				算法分为 **标记** 和 **清除** 两个阶段，首先标记出所有需要回收的对象

​				在标记完成后统一回收所有

##### 			缺点

​				效率低、产生内存碎片，碎片太多会导致提前 GC	

#### ​		复制算法						

##### 			算法

​				将可用对象复制从 幸存0区复制到幸存1区，再将这个幸存0区清除掉

##### 			优缺点

​				高效无内存碎片，占用两份内存空间

#### 		标记整理

##### 			算法

​				**标记** -> 将所有存活对象向一端移动，清除掉另一端的空间

#### ​		分代GC

​			年轻代用复制算法

​			老年代用标记清除、标记整理

#### 		对象分配

​			对象优先在年轻代分配

​			大对象直接进入老年代， -XX:PretenureSizeThreshold

​			长期存活对象进入老年代，-XX:MaxTenuringThreshold  

### 	GC 种类

#### 		串行收集器 Serial : Serial 、Serial Old

#### 		并行收集器 Parallel : Parallel Scavenge、Parallel Old 吞吐量	

​			多条 GC 线程并行工作，但此时用户线程处于等待状态

​			适合科学计算、后台处理等弱交互场景

​			吞吐量优先

```shell
-XX:+UseParallelGC  
-XX:+UseParallelOldGC
-XX:ParallelGCThreads=<N> # GC 线程数 CPU>8 N=5/8 CPU<8 N=CPU
# 其余对各个逻辑空间的内存大小设置等参数不详细记载
```

​			Server 模式下的默认收集器（内存大于2G 的服务器，默认都是 Server模式）

#### 		并发收集器 Concurrent : CMS、G1，停顿时间

​			指用户线程与GC线程同时执行（可能交替执行）

​			GC 线程不会影响用户线程的运行，适用 Web

​			响应时间优先

```shell
CMS : -XX:+UseConcMarkSweepGC   -XX:+UseParNewGC

G1: -XX:+UseG1GC

# CMS 垃圾收集过程
CMS initial mark : 初始标记 Root,STW
CMS concurrent mark : 并发标记
CMS-concurrent-preclean : 并发预清理

# CMS 的缺点
CPU 敏感
浮动垃圾
空间碎片
-XX:ConcGCThreads # 并发的 GC 线程数
-XX:+UseCMSCompactAtFullCollection # FullGC 之后做压缩
-XX:CMSFullGCsBeforeCompaction # 多少次 FullGC 之后压缩
-XX:CMSInitiatingOccupancyFraction # 触发 FullGC
```

​			

#### 		停顿时间 VS 吞吐量

##### 			停顿时间

​				GC 时中断应用执行的时间 -XX:MaxGCPauseMilis

##### 			吞吐量

​				GC 和应用的时间占比 -XX:GCTimeRatio=<n>，GC 占 1/1+n

#### 		如何选择垃圾回收器

​			优先调整堆的大小让服务器自己来选择

​			如果内存小于 100M，使用串行收集器

​			