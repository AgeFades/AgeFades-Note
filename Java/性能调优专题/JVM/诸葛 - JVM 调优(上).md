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

