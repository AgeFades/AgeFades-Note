[TOC]

# 诸葛 - JVM 调优(上)

## Jmap

```shell
# 用来查看 Java 进程 内存信息、实例个数 及 占用内存大小。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606327846.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606337934.png)

```shell
# num : 序号

# instances : 实例数量

# bytes : 占用空间大小

# class name : 类名称
	# [C  is a char[]
	# [S is a shot[]
	# [I is a int[]
	# [B is a byte[]
	# [[i is a int[][]
```

### 堆信息

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606473864.png)

### 堆内存 dump

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606497978.png)

```shell
# 也可以设置 OOM 自动导出 dump 文件（内存很大的时候，可能会导不出来）
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=路径
```

### Jvisualvm 导入 dump

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606626570.png)

## Jstack

```shell
# 用 jstack + 进程id 查找死锁:
```

```java
public class DeadLockTest {
  private static Object lock1 = new Object();
  private static Object lock2 = new Object();
  
  public static void main(String[] args) {
    
    new Thread(() -> {
      synchronized (lock1) {
        try {
          System.out.println("thread1 begin");
          Thread.sleep(5000);
        } catch (InterruptedException e) {
          
        }
        synchronized (lock2) {
          System.out.println("thread1 end");
        }
      }
    }).start();
    
    new Thread(() -> {
      synchronized (lock2) {
        try {
          System.out.println("thread2 begin");
          Thread.sleep(5000);
        } catch (InterruptedException e) {
          
        }
        synchronized (lock1) {
          System.out.println("thread2 end");
        }
      }
    }).start();
   
    System.out.println("main thread end");
  }
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606925200.png)

### Jvisualvm 自动检测死锁

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596606954581.png)

### Jstack 找出占用 CPU 最高的线程堆栈信息

```java
public class Math {
  public static final int initData = 666;
  public static User user = new User();
  
  public int compute() {
    int a = 1;
    int b = 2;
    int c = (a + b) * 10;
    return c;
  }
  
  public static void main(String[] args) {
    Math math = new Math();
    while (true) {
      math.compute();
    }
  }
  
}
```

```shell
# 使用 top -p <pid>，显示 Java 进程的内存情况
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607203317.png)

```shell
# 按 H，获取每个线程的内存情况
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607248300.png)

```shell
# 找到占用资源最高的 pid，转为 16 进制，比如为 4cd0
```

```shell
# 执行 jstack <pid> | grep -A 10 4cd0
	# 得到线程堆栈信息中 4cd0 线程所在行的后10行
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607397788.png)

## Jinfo

```shell
# 查看正在运行的 Java 应用程序的扩展参数
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607486231.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607501878.png)

## Jstat

```shell
# 查看 堆内存各部分的使用量、以及加载类的数量
```

### 垃圾回收统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607648347.png)

```shell
# S0C: 第一个幸存区的大小，单位 KB

# S1C: 第二个幸存区的大小

# S0U: 第一个幸存区的使用大小

# S1U: 第二个幸存区的使用大小

# EC: Eden 区的大小

# EU: Eden 区的使用大小

# OC: Old 区的大小

# OU: Old 区的使用大小

# MC: 方法区大小（元空间）

# MU: 方法区使用大小

# CCSC: 压缩类空间大小

# CCSU: 压缩类空间使用大小

# YGC: 年轻代垃圾回收次数

# YGCT: 年轻代垃圾回收消耗时间，单位 s

# FGC: 老年代垃圾回收次数

# FGCT: 老年代垃圾回收消耗时间，单位 s

# GCT: 垃圾回收消耗总时间，单位 s
```

### 堆内存统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596607970400.png)

```shell
# NGCMN: 新生代最小容量

# NGCMX: 新生代最大容量

# NGC: 当前新生代容量

# S0C: 第一个幸存区大小

# S1C: 第二个幸存区的大小

# EC: Eden 区的大小

# OGCMN: 老年代最小容量

# OGCMX: 老年代最大容量

# OGC: 当前老年代大小

# OC: 当前老年代大小

# MCMN: 最小元数据容量

# MCMX: 最大元数据容量

# MC: 当前元数据空间大小

# CCSMN: 最小压缩类空间大小

# CCSMX: 最大压缩类空间大小

# CCSC: 当前压缩类空间大小

# YGC: 年轻代 gc 次数

# FGC: 老年代 gc 此处
```

### 新生代垃圾回收统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371700711.png)

### 新生代内存统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371730324.png)

### 老年代垃圾回收统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371750284.png)

### 老年代内存统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371779766.png)

### 元数据空间统计

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371797870.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597371827038.png)

## JVM 运行情况预估

```shell
# 用 jstat -gc pid 命令可以得到一些关键数据，
	# 然后采用之前的一些优化思路， 
	
	# 先做一些初始化的 JVM 参数设置。
	
	# 比如:
		# 堆内存大小
		# 年轻代大小
		# Eden 和 Survivor 的比例
		# 老年代的大小
		# 大对象的阈值
		# 大龄对象进入老年代的阈值等
```

### 年轻代对象增长的速率

```shell
# 用 jstat -gc pid 1000 10（每隔 1s 执行 1次命令，共执行 10 次），
	# 观察 Eden 区的使用，估算每秒 eden 大概新增多少对象，
	
	# 需要在不同的时间（高峰期、平淡期）估算
```

### Young GC 的触发频率和每次耗时

```shell
# 知道年轻代对象增长速率后，就能根据 eden 区的大小推算出 Young GC 大概多久触发一次

# Young GC 的平均耗时可以通过 YGCT/YGC 公式算出，
	# 根据结果，大概就只能知道 系统多久会因为 Young GC 执行而卡顿多长时间。
```

### 每次Young GC 后有多少对象存活和进入老年代

```shell
# 上面已经大概知道Young GC 的频率，假设是每分钟一次，
	# jstat -gc pid 60000 10 (每隔 60s 执行一次命令，共执行 10 次)
	
	# 观察每次结果 eden、survivor、old 使用的变化情况，
	
	# 在每次 Young GC 后，eden 区使用一般会大幅减少，survivor 和 old 都可能增长，
	
	# 这些增长的对象就是每次 Young GC 后存活的对象，
	
	# 同时还可以看出每次 Young GC 进 Old 大概多少对象，从而推算出 老年代对象增长速率
```

### Full GC 的触发频率和每次耗时

```shell
# 知道了 老年代对象 的增长速率，就可以推算出 Full GC 的触发频率

# Full GC 的每次耗时公式: FGCT/FGC
```

### 优化思路

```shell
# 简单来说，尽量让每次 Young GC 后的存活对象，小于 Survivor 区域的 50%，

	# 尽量让对象在年轻代完成 创建 -> GC，从而减少进入老年代，降低 Full GC 频率。
```

## 频繁 Full GC 案例

### 配置条件

```shell
# 机器配置: 2核4G

# JVM内存大小: 2G

# 系统运行时间: 7天

# 期间发生的 Full GC 次数和耗时: 500多次，200多秒

# 期间发生的 Young GC 次数耗时: 1W多次，500多秒

# 平均每天 70 多次 Full GC，每次 Full GC 耗时 400ms 左右

# 平均每天 1000 都次 Young GC，每次 Young GC 耗时 50ms 左右
```

### JVM 参数

```shell
-Xms1536M -Xmx1536M -Xmn512m -Xss256K -XX:SurvivorRatio=6 -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:CMSInititatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597818212122.png)

### 原因分析

```shell
# 根据前文 对象挪动到老年代 的规则，进行推理，

	# 经过分析，感觉像是由于 动态年龄判断机制 导致 full gc 较为频繁。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597818348534.png)

### 第一次参数调优

```shell
-Xms1536M -Xmx1536M -Xmn1024M -Xss256K -XX:SurvivorRatio=6 -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M -XX:+UseParNewGC -XX:+UseConcMarkSweepGC 
-XX:CMSInitiatingOccupancyOnly
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597819292312.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597819319624.png)

```shell
# 优化完发现没什么变化，甚至 Full GC 的次数比 Young GC 次数还多了。

# 推测下 Full GC 比 Young GC 还多的原因是什么。
	# 1. 元空间不足导致的多余的 Full GC
	
	# 2. 显示调用 System.gc() 造成多余的 Full GC
		# 这种一般线上尽量通过 +XX:+DisableExplicitGC 参数禁用，
		
		# 如果加上 JVM 启动参数，那么代码中调用 System.gc() 没有任何效果
		
	# 3. 老年代空间分配担保机制
	
# 大概分析原因之后，我们可以使用 jmap 看下大概是哪些对象大量产生和占据空间
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597819591826.png)

```shell
# 发现 User 对象大量产生，接下来需要找到对应的代码确认。

# 如果直接去项目里查，可能会发现很多处创建 User 对象，无法精准定位。

# 使用 jstack 分析定位 cpu 占用率高的代码，定位到具体一直创建大量 User 对象的地方。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1597819727895.png)

```shell
# 至此，发现问题所在，解决代码问题即可。
```

## 内存泄漏是怎么回事

```shell
# 比如说，JVM 项目自己维护一个静态 HashMap，不断往里面存放缓存数据，
	# 没考虑到这个 HashMap 的容量释放问题，
	
	# 一定时间过后，触发 Full GC，发现无法清除该 HashMap 对象，抛出 OOM
	
	# 这就是一种内存泄漏: 对一个永远无法 GC 的对象，不断插入对象数据，而不考虑释放。
```

