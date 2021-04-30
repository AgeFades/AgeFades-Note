[TOC]

# 诸葛 - JVM对象创建内存分配机制详解

## 对象的创建

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596435835309.png)

### 类加载检查

```shell
# JVM 遇到一条 new 指令时，
	# 首先将去检查这个指令的参数，是否能在常量池中定位到一个类的符号引用，
	# 并且检查这个符号引用代表的类是否已被 加载、解析、初始化 过，
	# 如果没有，那必须先执行相应的类加载过程。
	
# new 指令对应到语言层面上讲是，new关键词、对象克隆、对象序列化等。
```

### 分配内存

```shell
# 在类加载检查通过后，接下来 JVM 将为新生对象分配内存。

# 对象所需内存的大小在 类加载 完成后便可以完全确定，
	# 为对象分配空间的任务等同于 = 把一块确定大小的内存从 JVM 堆中划分出来。
	
	# 这个步骤有两个问题:
		# 如何划分内存
		
		# 在并发情况下，可能出现正在给对象 A 分配内存，指针还没来得及修改，
			# 对象 B 又同时使用了原来的指针来分配内存的情况。
```

#### 划分内存的方法

##### 指针碰撞（Bump the Pointer: 默认）

```shell
# 如果 JVM 堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边
	# 中间放着一个指针作为分界点的指示器
	
	# 那所分配内存就仅仅是把那个指针指向空闲空间、挪动一段与对象大小相等的距离
```

##### 空闲列表（Free List）

```shell
# 如果 JVM 堆中的内存并不是规整的，已使用的内存和空闲的内存相互交错
	# 那就没有办法简单地进行指针碰撞了
	
	# JVM 就必须维护一个列表,记录上哪些内存是可用的
	
	# 在分配时,从列表中找到一块足够大的空间分配给对象实例,并更新列表上的记录
```

#### 解决并发问题的方法

##### CAS（compare and swap）

```shell
# JVM 采用 CAS + 失败重试 的方法保证更新操作的原子性，
	# 来对分配内存空间的动作进行同步处理
```

##### 本地线程分配缓冲（Thread local Allocation Buffer，TLAB: 默认开启）

```shell
# 把 内存分配 的动作按照 线程 划分在不同的空间之中进行，
	# 即 每个线程 在 JVM 堆中预先分配一小块内存，
	
	# 通过 -XX:+UseTLAB 参数来设定 JVM 是否使用 TLAB
	
	# -XX:TLABSize 指定 TLAB 大小。
```

### 初始化

```shell
# 内存分配完成后，JVM 需要将分配到的内存空间都初始化为 零 值（不包括对象头），
	# 如果使用 TLAB，这一工作过程也可以提前至 TLAB 分配时进行。
	
	# 这一步操作保证了对象的实例字段在 Java 代码中可以不赋初始值就直接使用，
	
	# 程序能访问到这些字段的数据类型所对应的 零 值 
```

### 设置对象头

```shell
# 初始化 零 值之后，JVM 要对对象进行必要的设置，
	# 例如: 这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC年龄等
	
	# 这些信息存放在对象的对象头 Object Header 之中
	
# 在 HotSpot JVM 中，对象在内存中存储的布局可以分为 3 块区域，
	# 对象头(Header)、实例数据(Instance Data)、对其填充(Padding)
	
# HotSpot JVM 的对象头包含两部分信息:
	# 1. 用于存储对象自身的运行时数据
		# 如 HashCode、GC年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳 等
		
	# 2. 类型指针
		# 即 对象 指向 类元数据 的指针，JVM 通过这个指针来确定这个对象是哪个类的实例.
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596437908304.png)

```shell
# 对象头在 Hotspot 的 C++ 源码里的注释如下:
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596439250796.png)

### 执行`<`init`>`方法

```shell
# 执行 <init> 方法，即对象按照程序员的意愿进行初始化。
	# 对应到语言层面上讲，就是为属性赋值（和上面的赋零值不同，这是由程序员赋的值）
		# 和执行构造方法。
```

## 对象大小与指针压缩

```shell
# 对象大小可以用 jol-core 包查看，引入依赖:
```

```xml
<dependency>
	<groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.9</version>
</dependency>
```

```java
import org.openjdk.jol.info.ClassLayout;
/**
 * 计算对象大小
 */
public class JOLSample {

  public static void main(String[] args) {
    ClassLayout layout = ClassLayoout.parseInstance(new Object());
    System.out.println(layout.toPrintable());
    System.out.println();
    
    ClassLayout layout1 = ClassLayoout.parseInstance(new int[]{});
    System.out.println(layout1.toPrintable());
    System.out.println();
    
    ClassLayout layout2 = ClassLayoout.parseInstance(new A());
    System.out.println(layout2.toPrintable());
    System.out.println();
  }
  
 /**
  * -XX:+UseCompressedOops : 默认开启的压缩所有指针
  * -XX:+UseCompressedClassPointers : 默认开启的压缩对象头里的类型指针 Klass Pointer
  * Oops : Ordinary Object Pointers
 	*/
 static class A {
   
   /**
    * 8B mark word
  	* 4B Klass Pointer 
  	*		如果关闭压缩 -XX:-UseCompressedClassPointers 
  	* 	或 -XX:-UseCompressedOops, 则占用 8B
  	*/
   int id;
   
   String name;
   
   byte b;
   
   Object o;
 }
  
}
```

### 运行结果

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596440441839.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596440450929.png)

### 什么是 Java 对象的指针压缩

```shell
# JDK1.6 开始，在 64位 的操作系统中，JVM 支持指针压缩

# JVM 配置参数: UseCompressedOops
	# conpressed : 压缩
	# oop : 对象指针
	# 默认启用
```

### 为什么要进行指针压缩

```shell
# 1. 在 64 位平台的 HotSpot 中使用 32 位指针，内存使用会多出 1.5 倍左右
	# 使用较大指针在主内存和缓存之间移动数据，占用较大宽带，同时 GC 也会承受较大压力。
	
# 2. 为了减少 64 位平台下内存的消耗，启用指针压缩功能

# 3. 在 JVM 中，32位地址最大支持 4G 内存 (2的32次方)
	# 可以通过对对象指针的压缩编码、解码方式进行优化，
	
	# 使 JVM 只用 32 位地址就可以支持更大的内存配置 (小于等于32G)
	
# 4. 堆内存小于 4G 时，不需要启用指针压缩，JVM 会直接去除高 32 位地址，即使用低虚拟地址空间

# 5. 堆内存大于 32G 时，压缩指针会失效，会强制使用 64位（即 8字节）
	
# 6. 指针压缩是否使用、取决于堆内存的大小，实际上是取决于 对象寻址所需存储的比特位
```

## 对象内存分配

### 流程图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596442158877.png)

### 对象栈上分配

```shell
# 通过 JVM 内存分配得知，Java 中的对象都是在堆上进行分配
	# 当对象没有被引用的时候，需要依靠 GC 进行回收内存，
	
	# 如果对象数量较多的话，会给 GC 带来较大压力，也间接影响了应用的性能。
	
	# 为了减少 临时对象 在堆内分配的数量，JVM 通过 逃逸分析 确定该对象会不会被外部访问，
		# 如果不会逃逸，可以将该对象在 栈上分配 内存，
		
		# 这样该对象占用的内存空间就可以随栈帧出栈而销毁，减轻 GC 压力。
		
# 对象逃逸分析:
	# 就是分析对象动态作用域，当一个对象在方法中被定义后，
	
	# 它可能被外部方法所引用，例如作为 调用参数 传递到其他地方中。
```

```java
public User test1() {
  User user = new User();
  user.setId(1);
  user.setName("Swordsman");
  return user;
}

public void test2() {
  User user = new User();
  user.setId(1);
  user.setName("zhuge");
}
```

```shell
# 很显然，test1 方法中的 user 对象被返回了，这个对象的作用域范围不能确定。

# test2 方法中的 user 对象，可以确定当方法结束时，这个对象就是无效对象了，
	# 对于这样的对象，我们可以将其分配在栈内存里，让其随方法结束时，跟随栈内存一起被回收掉。
	
# JVM 对于这种情况，可以通过开通 逃逸分析参数(-XX:+DoEscapeAnalysis) 来优化对象内存分配位置
	# 使其通过 标量替换 优先分配在栈上(栈上分配)
	
	# JDK7 之后默认开启逃逸分析
	
# 标量替换:
	# 通过 逃逸分析 确定该对象不会被外部访问，并且对象可以被进一步分解时，
		# JVM 不会创建该对象，而是将该对象 成员变量 分解若干个被这个方法使用的成员变量所代替
		
		# 这些代替的成员变量在 栈帧 或 寄存器 上分配空间，
		
		# 就不会因为没有一大块连续空间导致对象内存不够分配，
		
		# JDK7之后默认开启 标量替换 (-XX:+EliminateAllocations)
		
# 标量与聚合量:
	# 标量即不可被进一步分解的量，Java 的基本数据类型就是标量（int、long、reference...）
	
	# 聚合量是可以被进一步分解的量，Java 中的对象就是可以被进一步分解的聚合量
```

#### 栈上分配示例代码

```java
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.date.TimeInterval;
import lombok.Data;

/**
 * 栈上分配、标量替换
 * 代码调用了1亿次alloc()，如果是分配到堆上，堆空间不够，必然会触发大量GC。
 *
 * 使用如下参数不会大量发生GC
 * -Xmx15m -Xms15m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations 
 * 
 * 使用如下参数都会发生大量GC
 * -Xmx15m -Xms15m -XX:‐DoEscapeAnalysis -XX:+PrintGC -XX:+EliminateAllocations
 * -Xmx15m -Xms15m -XX:+DoEscapeAnalysis -XX:+PrintGC -XX:‐EliminateAllocations
 * @author DuChao
 * @date 2020/7/29 5:00 下午
 */
public class AllotOnStack {

    public static void main(String[] args) {
        TimeInterval timer = DateUtil.timer();
        for (int i = 0; i < 100000000; i++) {
            alloc();
        }
        System.out.println("耗时: " + timer.interval() + "毫秒");
    }

    private static void alloc() {
        User user = new User();
        user.setName("张三");
        user.setPhone("13812345678");
    }

    @Data
    static class User {
        private String name;
        private String phone;
    }

}

```

#### 结论

```shell
# 栈上分配依赖于 逃逸分析 和 标量替换
```

### 对象在 Eden 区分配

```shell
# 大多数情况下，对象在新生代中 Eden 区分配，
	# 当 Eden 区没有足够空间进行分配时，JVM 将发起一次 Minor GC。
```

#### Minor GC 和 Full GC 有什么不同？

```shell
# Minor GC: 
	# 指发生在新生代的垃圾收集动作，Minor GC 非常频繁，回收速度一般也比较快

# Full GC: 
	# 一般回收老年代、年轻代、方法区的垃圾，速度一般会比 Minor GC 慢 10 倍以上
```

#### Eden 与 Survivor

```shell
# 大量的对象被分配在 Eden 区，Eden 区满了之后，会触发 Minor GC，
	# 垃圾对象将被回收，剩余存活对象会被 复制 到空的那块 Survivor 区，
	
	# 下次 Eden 区满了后，又会触发 Minor GC，把 Eden 区和 Survivor 区垃圾对象回收，
		# 将剩余存活对象 复制 到另一块空的 Survivor 区。
		
# 因为 Eden 区大多数对象都是 朝生夕死，存活时间很短，
	# 所以 JVM 默认的 8:1:1 比例是很合适的，
	
	# 让 Eden 区尽量大，Survivor 区够用即可。
	
# JVM 默认会有 -XX:+UseAdaptiveSizePolicy ，导致 8:1:1 比例自动调整
	# 如果不想让其调整，关闭参数即可
```

### 对象在 Old 区分配

#### 大对象直接进入老年代

```shell
# 大对象就是需要大量连续内存空间的对象，
	# 比如: 字符串、数组...
	
# JVM 参数 -XX:PretenureSizeThreshold 可以设置大对象的大小，
	# 如果对象超过设置大小，会直接跳过年轻代、进入老年代，
	
	# 这个参数只在 Serial 和 ParNew 两个收集器下有效。
	
# JVM 这样做的意义是: 
	# 避免 Minor GC 复制操作时，为大对象分配内存的性能、吞吐量损耗。
```

#### 长期存活的对象将进入老年代

```shell
# 既然 JVM 采用了 分代收集 的思想来管理内存，
	# 那么内存回收时，就必须能识别 对象 是放在 新生代 还是 老年代。
	
# JVM 给每个 对象 一个对象年龄计数器（Age），
	# 对象每经历一次 Minor GC 并存活，对象年龄加1
	
	# 默认对象年龄到 15 岁时，进入老年代（CMS 收集器默认6岁）
	
	# 晋升老年代的年龄阈值，可以通过参数 -XX:MaxTenuringThreshold 来设置。
```

#### 对象动态年龄判断

```shell
# 当前放对象的 Survivor 区，
	# 一批对象的总大小 大于一块 Survivor 区域内存大小的 50% 
		# (可通过 -XX:TargetSurvivorRatio 指定)，
    
   # 那么，此时 大于等于 这批对象 年龄最大值 的对象，就会直接被挪到老年代。
   

# 举例:
	# Survivor 区域里现在有一批对象，年龄1 + 年龄2 + 年龄n 的多个年龄对象总和
		# 超过了 Survivor 区域的 50%，此时就会把 年龄n及以上 的对象都挪到老年代。
		
# JVM 这样做的意义是:
	# 希望那些可能是长期存活的对象，尽早进入老年代，不要占用年轻代空间，损耗复制性能
	
# 对象动态年龄判断一般都是在 Minor GC 之后触发的。
```

#### 老年代空间分配担保机制

```shell
# 年轻代每次 Minor GC 之前， JVM 都会计算下老年代剩余可用空间，
	# 如果这个可用空间 小于 年轻代里 现有的所有对象 大小之和,
	
	# 就会检查是否开启 -XX:HandlerPromotionFailure（JDK8 默认开启），
	
	# 如果开启了，就会看 老年代的可用内存大小，
		# 是否 大于 之前 Minor GC 后进入老年代的对象的平均大小
		
			# 如果大于的话，不会直接 Full GC，如果 Minor GC 完剩余对象大小 > 老年代内存空间，
				# 也会触发 Full GC
		
			# 如果 小于 的话，就会触发一次 Full GC，对 老年代 和 年轻代 一起做一次 GC
		
		# 如果 GC 完了，还是没有足够空间存新的对象，就会触发 OOM
		
	# 如果没有开启，直接 Full GC
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596506918128.png)

## 对象内存回收

```shell
# 堆中几乎放着所有的对象实例，
	# 对堆垃圾回收前的第一步就是要判断哪些对象是垃圾对象，
	
	# 即不能再被任何途径使用的对象。
```

### 引用计数法

```shell
# 在对象中添加一个引用计数器，每当有一个地方引用该对象，计数器就 +1，
	# 当引用失效，计数器 -1，
	
	# 任何时候，计数器为 0 的对象就是垃圾对象。
	
# 引用计数法实现简单、效率高，
	# 但是 JVM 并没有选择该算法来确认垃圾对象，
	
	# 最主要的原因就是它很难解决对象之间 相互循环引用 的问题。
	
# 所谓对象循环引用，
	# 如下 A、B 所示，除了 A、B 相互引用，
	
	# 其余再无任何引用指向这两个对象，
	
	# 导致 A、B 计数器永远不为0，所以不能被回收。
```

```java
public ReferenceCountingGc {
  Object instance = null;
  
  public static void main(String[] args) {
    ReferenceCountingGc A = new ReferenceCountingGc();
    ReferenceCountingGc B = new ReferenceCountingGc();
		A.instance = B;
    B.instance = A;
    A = null;
    B = null;
  }
}
```

### 可达性分析算法

```shell
# 将 GC Roots 对象作为起点，向下遍历引用对象，
	# 遍历到的对象都标记为 非垃圾对象，
	
	# 其余未标记到的对象都是垃圾对象。
	
# GC Roots:
	# 线程栈的局部变量
	
	# 方法区的静态变量
	
	# 方法区的常量
	
	# 本地方法栈的变量
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1596507442020.png)

### 常见引用类型

```shell
# Java 引用类型一般分为如下四种。
```

#### 强引用

```java
public static User user = new User();
```

#### 软引用

```shell
# 将对象用 SoftReference 软引用类型的对象包裹，
	# 正常情况下不会被回收，
	
	# 但是 GC 完成后，发现不能释放出足够的空间来存放新的对象时，
		# 会把软引用对象回收掉。
		
# 软引用可用来实现内存敏感的高速缓存。

# 软引用在实际中的应用场景:
	# 例如: 浏览器的后退按钮
```

```java
public static SoftReference<User> user = new SoftReference<User>(new User());
```

#### 弱引用

```shell
# 将对象用 WeakReference() 弱引用类型的对象包裹，
	# 弱引用和没有引用差不多,GC 会直接回收掉，很少用。
```

```java
public static WeakReference<User> user = new WeakReference<User>(new User());
```

#### 虚引用

```shell
# 虚引用是最弱的一种引用关系，几乎不用。
```

### finalize()

```shell
# finalize() 方法用来最终判定对象是否存活。

# 即时在可达性分析算法中、不可达的对象也并发是一定会被 GC 回收的。
	# 此时该对象还只经历过一次标记、要真正 GC 对象、至少要经历再次标记的过程。
	
	# 标记的前提是，对象在进行可达性分析后发现没有与 GC Roots 想连接的引用链。
	
# 1. 第一次标记并进行一次筛选
	# 筛选的条件是此对象 是否有必要 执行 finalize() 方法
	
	# 当对象没有覆盖 finalize() 方法，对象将被直接回收。
	
# 2. 第二次标记
	# 如果该对象覆盖了 finalize() 方法，
	
	# 该对象可以在 finalize() 中与引用链上的任意一个对象建立关联、
		# 即可在 第二次标记 时，将它移除出 "垃圾对象" 的集合。
		
	# 一个对象的 finalize() 方法只会被执行一次。
	
# 该方法只需了解即可，基本没有实际运用场景。
```

### 判断 "无用" 类

```shell
# Full GC 中，方法区主要回收的是 "无用" 的类。

# 类 需要同时满足 3 个条件才能算是 "无用" 的类:
	# 该类的 所有实例 已经被回收
	
	# 加载该类的 ClassLoader 已经被回收
	
	# 该类对应的 java.lang.Class 对象没有任何引用，无法在任何地方通过反射访问该类的方法。
```

