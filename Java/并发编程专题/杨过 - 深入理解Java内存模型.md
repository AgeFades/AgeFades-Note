[TOC]

# 杨过 - 深入理解Java内存模型

## 什么是 JMM 模型

```shell
# Java Memory Model、抽象概念:
	# 描述一组 规则或规范，
		
	# 定义 各个变量（包括 实例字段、静态字段 和 构成数组对象的元素）的访问方式。
	
# JVM 运行程序的实体是 线程，而每个 线程 创建时，
	# JVM 都会为其创建一个 工作内存（或称为 栈空间），用于存储 线程私有数据，
	
	# 而 Java 内存模型中规定，所有变量 都存储在 主内存，主内存是 共享内存区域，
	
	# 所有线程都可以访问，但 线程对变量的操作（读、取、赋值等）必须在 工作内存 中进行。
	
# 首先将 变量 从 主内存 拷贝到 线程自己的工作空间，
	# 然后对变量进行操作，
	
	# 操作完成后再将 变量 写回 主内存。
	
# 因为 线程 不能直接操作 主内存 中的变量，
	# 线程 工作内存 中存储着 主内存的变量副本拷贝，
	
	# 而 线程工作内存 是每个线程的私有区域数据，
	
	# 所以 不同的线程间 无法访问对方的 工作内存，
	
	# 线程间的通信（传值）必须通过主内存来完成。
```

### JMM 不同于 JVM 内存区域模型

```shell
# JMM 与 JVM 内存区域的划分，是不同的概念层次，
	# JVM 描述的是一组 规则，
	
	# 通过这组 规则 控制程序中 各个变量 在 共享数据区域 和 私有数据区域 的访问方式，
	
	# JMM 是围绕 原子性、有序性、可见性 展开。
	
# JMM 与 Java 内存区域唯一相似点，
	# 都存在 共享数据区域 和 私有数据区域，
	
	# 在 JMM 中，主内存 属于 共享数据区域，从某个程度上讲，包括了 堆和方法区，
	
	# 而 私有数据区域，包括 程序计数器、虚拟机栈、本地方法栈。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602646146551.png)

#### 主内存

```shell
# 主要存储 Java 实例对象，所有线程创建的 实例对象 都存放在 主内存 中，
	# 不管该 实例对象 是 成员变量 还是 方法区 中的 本地变量（也称局部变量），
	
	# 也包括了 共享的类信息、常量、静态变量。
	
# 由于是 共享数据区域，多条线程 对 同一个变量 进行访问可能会发生 线程安全问题。
```

#### 工作内存

```shell
# 主要存储当前方法的所有本地变量信息（工作内存中存储着主内存中的变量副本拷贝），
	# 每个线程 只能访问自己的 工作内存，
	
	# 即: 线程中的 本地变量 对其他线程是不可见的，
	
	# 就算是 两个线程 执行 同一段代码，也会各自在自己的 工作内存 中创建 本地变量，
	
	# 也包含了 字节码行号指示器、相关 Native 方法的新信息。
	
# 由于是 私有数据区域，线程间无法相互访问工作内存，因此 工作内存数据 不存在 线程安全问题。
```

```shell
# 根据 JVM 虚拟机规范 主内存 与 工作内存 的 数据存储类型 以及 操作方式，
	# 对于一个 实例对象 中的 成员方法 而言，
	
	# 如果方法中包含本地变量是 基本数据类型（boolean/byte/short/char/int/long...）,
		# 将直接存储在 工作内存 的 栈帧结构 中。
	
	# 如果本地变量是 引用类型，该变量的引用会存储在 功能内存 的栈帧中，
		# 而 对象实例 将存储在 主内存（共享数据区域、堆）中。
		
	# 如果是 实例对象的成员变量，不管是 基本数据类型、包装类型、引用类型，
		# 都会被 存储 到堆区。
		
	# 如果是 static变量 以及 类本身相关信息 将会存储在 主内存 中。
	
# 注意:
	# 在 主内存 中的 实例对象，可以被 多线程 共享，
	
	# 假设 两条线程 同时调用了同一个对象的同一个方法，
	
	# 那 两条线程 会将要操作的数据拷贝一份到自己的工作内存中，
	
	# 执行完成操作后才 刷新到主内存。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602659072182.png)

#### Java内存模型与硬件内存架构的关系

```shell
# 多线程的执行 最终都会映射到 硬件处理器 上进行执行，
	# 但 Java内存模型 和 硬件内存架构 并不完全一致。
	
# 对 硬件内存 来说，只有 寄存器、缓存内存、主内存 的概念，
	# 并没有 工作内存（线程私有数据区域）和 主内存（堆内存）之分，
	
	# 也就是说 Java内存模型 中 内存的划分 对 硬件内存 没有任何影响，
		# 不管是 工作内存 还是 主内存，对硬件来说，都会存储在 计算器主内存 中，
		
		# 也可能存储到 CPU缓存 或 寄存器中。
		
# 总体来说，Java内存模型 和 计算机硬件内存架构 是 相互交叉 的关系，
	# 是一种 抽象概念划分 与 真实物理硬件 的交叉，
	
	# 对于 Java内存区域划分 也是同样道理。
```

#### ![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602660855167.png)

### JMM 存在的必要性

```shell
# 线程安全问题:
	# 假设 A、B 两条线程同时读取 主内存中变量 a=1 到各自线程 工作内存 中,
	
	# A、B 修改 变量a 的值，再写回主内存，
	
	# 就可能造成 线程安全问题，比如 A.a = 2，B.a = 3
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602661113380.png)

```shell
# 以上关于 主内存 与 工作内存 之间的具体交互协议，
	# 即一个变量 如何从 主内存 拷贝到 工作内存、
	
	# 如果从 工作内存 同步到 主内存 之间的实现细节，
	
	# JMM 定义了以下八种操作来完成。
```

### 数据同步八大原子操作

```shell
1. lock(锁定): 
	# 例如: 线程A 标记锁定 主内存中的变量str
	
2. unlock(解锁):
	# 例如: 线程A 解除标记锁定 主内存中的变量str
	
3. read(读取):
	# 例如: 线程A 读取 主内存中的 变量str 到工作内存
	
4. load(载入):
	# 例如: 线程A 拷贝 变量str 到本地变量副本
	
5. use(使用):
	# 例如: 线程A 传输 变量str 到执行引擎
	
6. assign(赋值):
	# 例如: 线程A 接收 执行引擎返回str变量值，并赋值给 工作内存的变量str
	
7. store(存储):
	# 例如: 线程A 传输 变量str 到主内存
	
8. write(写入):
	# 例如: 线程A 传输 变量str值 到主内存，并修改原 变量str 值
```

```shell
# 主内存 -> 工作内存
	# 按顺序执行 read 和 load 操作。
	
# 工作内存 -> 主内存
	# 按顺序执行 store 和 write 操作。
	
# 但 JMM 只要求上述操作 必须按顺序执行，而 没有保证必须是连续执行。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602662234320.png)

#### 同步规则分析

```shell
# 1. 不允许一个线程 无原因地（没有发生过任何 assign 操作）把数据从 工作内存 同步回 主内存。

# 2. 一个新的变量只能在 主内存中 诞生，
	# 不允许在 工作内存中 直接使用一个 未被初始化（load 或 assign）的变量。
	
	# 即 对一个变量实施 use 或 store 操作之前，必须先自行 assign 和 load 操作。
	
# 3. 一个变量 在 同一时刻 只允许一条线程对其进行 lock 操作，
	# 但 lock 操作可以被 同一线程 重复执行多次，
	
	# 多次执行 lock 后，只有执行相同次数的 unlock 操作，变量才会被解锁。
	
	# lock 和 unlock 必须成对出现。
	
# 4. 对一个变量执行 lock 操作，将会清空 工作内存中 此变量的值，
	# 在 执行引擎 使用该变量之前，需要重新执行 load 或 assign 操作 初始化变量值。
	
# 5. 如果 一个变量 事先没有被 lock 锁定，则不允许对它执行 unlock 操作，
	# 也不允许去 unlock 一个被其他线程锁定的变量。
	
# 6. 对一个变量执行 unlock 操作之前，
	# 必须先把 此变量 同步到 主内存中（执行 store 和 write 操作）。
```

### 并发编程的可见性、原子性与有序性问题

#### 原子性

```shell
# 一个操作不可被中断，即使是 多线程环境下，一个操作一旦开始就不会被其他线程影响。

# 在 Java 中，对 基本数据类型 变量的 读取和赋值 操作 是 原子性的。
```

#### 可见性

```shell
# 一个线程 修改了某个 共享变量的值，其他线程能 马上感知修改后的值。

# 对于 串行程序来说，是不存在 可见性概念 的，因为只有一条线程按序执行。

# 多线程环境下，就出现了 可见性问题（写丢失问题）
	# 线程A、B 读取变量 a=1
	
	# 线程A 修改 a=2，write 回主内存
	
	# 线程B 修改 a=3，write 回主内存
	
	# 线程A 读取变量a，发现 a=3，而不是自己改的 a=2，这就是 写丢失问题
```

#### 有序性

```shell
# 多线程环境下，程序执行 可能出现 乱序现象，
	# 因为程序编译成 机器码指令 后，可能会出现 指令重排现象，
	
	# 重排后的指令 与 原指令 的顺序未必一致。
	
# Java 程序中，单线程内 所有操作都被视为 有序性行为，
	# 而 多线程环境下，一个线程 观察 另外一个线程，所有操作都是无序的。
```

### JMM 如何解决原子性&可见性&有序性问题

#### 原子性问题

```shell
# 除了 JVM 自身提供的 对基本数据类型读写 操作的原子性外，
	# 可以通过 synchronized 和 lock 实现原子性，
	
	# 因为 synchronized 和 lock 能够保证 任一时刻 只有一个线程访问该代码块。
```

#### 可见性问题

```shell
# volatile 保证 可见性。

# 当一个 共享变量 被volatile 修饰时，
	# 会保证 修改的值 立即被其他的线程看到，即 修改的值 立即更新到主存中，
	
	# 当其他线程需要读取时，它会去内存中读取新值。
	
# synchronized 和 lock 也可以保证可见性，
	# 因为它们可以保证 任一时刻 只有 一个线程 能访问共享资源，
	
	# 并在其释放锁之前，将修改的变量刷新到内存中。
```

#### 有序性问题

```shell
#  volatile 保证一定的 有序性。

# synchronized 和 lock 保证 有序性。
```

```shell
# 每个线程都有自己的工作内存（类似于 CPU 的高速缓存），
	# JMM 具备一些先天 "有序性"，即 不通过任何手段 就能保证 有序性，
	
	# 这通常被称为 happens-before 原则，
	
	# 如果 两个操作的执行顺序 无法从 happens-before 原则推导出来，
		# 就不能保证 它们的有序性，JVM 就可以随意地对它们进行重排序。
```

```shell
# Java 规范规定 JVM 线程内部 维持 顺序化语义。

# 即: 只要程序的最终结果与它顺序化情况的结果相等，
	# 那么 指令的执行顺序 可以与 代码顺序 不一致，
	
	# 这个过程叫做 指令的重排序。
	
# 指令的重排序意义是什么？
	#  JVM 能根据处理器特性（CPU多级缓存系统、多核处理器等），
	
	# 适当的对 机器指令 进行 重排序，
	
	# 使机器指令更符合 CPU 的执行特性，最大限度的发挥机器性能。
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602728550517.png)

#### as-if-serial语义

```shell
# 不管怎么重排序（编译器和处理器为了提高并行度），
	# （单线程）程序的执行结果不能被改变。
	
# 编译器、runtime和处理器，都必须遵守 as-if-serial 语义。

# 编译器和处理器 不会对存在 数据依赖关系的操作 做重排序，
	# 因为这种重排序可能会改变 执行结果。
	
	# 反之，不存在 数据依赖关系的操作，就可能被编译器和处理器重排序。
```

#### happens-before 原则

```shell
# 如果只靠 sychronized 和 volatile 来保证原子性、可见性、有序性，
	# 那做 并发程序 就相当麻烦。
	
# 从 JDK1.5 开始，Java使用新的 JSR-133 内存模型，
	# 提供 happens-before 来辅助保证 程序执行 的原子性、可见性、有序性问题，
	
	# 是判断 数据 是否存在竞争、线程是否安全的依据。
```

```shell
1. 程序顺序原则:
	# 单线程内必须保证 语义串行性。
	
2. 锁规则:
	# lock 必须在 unlock 之前（同一个锁）
	
3. volatile 规则:
	# volatile变量 在 每次被线程访问时，都会强制从 主内存 中读取该变量的值，
	
	# 而当变量发生变化时，又会强制将 最新的值 刷新到 主内存，
	
	# 所以，任何时刻 不同线程 总是能看到 volatile变量最新的值。
	
4. 线程启动规则:
	# thread.start() 方法优先于 线程的任何动作，
	
	# 即: thread1 在 thread2.start() 之前修改了共享变量的值，
	
	# thread2.start() 时，可以读到 thread1 修改的共享变量值。
	
5. 传递性:
	#  如果 A > B， B > C， 那么 A > C
	
6. 线程终止规则:
	# thread 所有操作都先于 线程终止。
	
	# thread.join() 方法的作用是，等待 当前执行的线程终止。

	# 假设 thread2 在终止之前，修改了共享变量，
		# thread1 从 thread2 的 join() 方法成功返回后，
		
		# thread2 对共享变量的修改将对 thread1 可见。
		
7: 线程中断规则:
	# 对 thread.interrupt() 方法的调用，
		# 先行发生于 被中断线程的代码 检测到 中断事件的发生，
		
		# 可以通过 thread.inteerrupted() 方法检测线程是否中断。
		
8: 对象终结规则:
	# 对象的构造函数执行，结束 先于 finalize() 方法。
```

## volatile 内存语义

```shell
# JVM 提供的轻量级 同步机制。

# 保证 可见性、禁止指令重排，不保证原子性。
```

### volatile 的可见性

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1602836185053.png)

```shell
# 使用 volatile 修饰的变量，用 javap 查看会发现，
	# 属性被额外标记为: ACC_VOLATILE
	
	# 这个标记会被 JVM 做额外处理
```

```shell
# 并不是说，不使用 volatile 或者 synchornized 或者 lock，
	# 线程之间 对 共享变量的操作就永远互不可见了。
	
# volatile 只是保证了及时性，即 happens-before 中的 volatile规则
```

```shell
# 关于 volatile 这里的笔记、代码不如周阳老师讲的

# 请参考本项目中 Java/Java底层/周阳 - Java底层上
```

```java
import java.util.concurrent.TimeUnit;

public class VolatileDemo {

    public static void main(String[] args) {
        User user = new User();
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t come in");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            user.add();
            System.out.println(Thread.currentThread().getName() + "\t num value : " + user.num);
        }, "SyncThread").start();

        while (user.num == 0) {}
        System.out.println(Thread.currentThread().getName() + "\t over,num value : " + user.num);
    }

}

class User {
    volatile int num = 0; // 去掉 volatile 试试
    void add() {
        this.num = 60;
    }
}
```

### volatile 无法保证原子性

```java
public class VolatileDemo {

    public static void main(String[] args) {
        User user = new User();
        for(int i = 1;i <= 20; i++ )
        {
            new Thread(() -> {
                for (int j = 1; j <= 1000; j++) {
                    user.add();
                }
            },String.valueOf(i)).start();
        }
        while (Thread.activeCount() > 2){
            Thread.yield();
        }

        System.out.println(Thread.currentThread().getName() + "\t over,num value : " + user.num);
    }
}

class User {
    volatile int num = 0;
    void add() {
        num++;
    }
}
```

```shell
# 想解决可以用 synchronized 或 lock 或 atomic
```

### volatile 禁止重排优化

```shell
# volatile 通过 内存屏障（Memory Barrier）实现禁止指令重排。
```

### 硬件层的内存屏障

```shell
# Intel 硬件提供了系列 内存屏障:

1. lfence
	# Load Barrier 读屏障
	
2. sfence
	# Store Barrier 写屏障
	
3. mfence
	# 全能型屏障，具备 ifence 和 sfence 能力
	
4. Lock前缀
	# 不是一种内存屏障，但能完成类似 内存屏障 的功能。
	
	# Lock 会对 CPU总线 和 高速缓存加锁，可以理解为 CPU指令集 的一种锁。
	
	# 后面可以跟: ADD、ADC、AND、BTC、BTR、BTS ... 等等指令
```

```shell
# 不同硬件实现 内存屏障 的方式不同，JMM 屏蔽了底层硬件差异，
	# 由 JVM 来为 不同的平台 生成 相应的机器码。
	
# JVM 提供 四类内存屏障指令:
```

| 屏障类型   | 指令示例                   | 说明                                                         |
| ---------- | -------------------------- | ------------------------------------------------------------ |
| LoadLoad   | Load1; LoadLoad; Load2     | 保证 load1 的读取操作在 load2 及后续读取操作之前执行         |
| StoreStore | Store1; StoreStore; Store2 | 在 store2 及其后的写操作执行前，保证 store1 的写操作已刷新到主内存 |
| LoadStore  | Load1; LoadStore2; Store2; | 在 Store2 及其后的写操作执行前，保证 load1 的读操作已读取结束 |
| StoreLoad  | Store1; Storeload; Load2   | 保证 store1 的写操作已刷新到主内存之后，load2 及其后的读操作才能执行 |

```shell
# 内存屏障，又称 内存栅栏，是一个 CPU指令。

# 作用:
1. 保证特定操作的执行顺序

2. 保证某些变量的内存可见性（volatile 就是利用该特性实现的 内存可见性）

# 因为 编译器和处理器 都能执行 指令重排优化，
	# 所以在 指令间 插入一条 Memory Barrier 则会告诉 编译器和CPU，
	
	# 不管什么指令，都不能和这条 Memory Barrier 指令重排序，
	
	# 即: 通过插入内存屏障 禁止在 内存屏障前后的指令执行重排序优化。
	
# Memory Barrier 还可以 强制刷出 各种CPU的缓存数据，
	# 所以，任何CPU上的线程 都能读取到 这些数据的最新版本。
	
# 所以，volatile 关键字 实现 可见性、禁止重排优化，就是靠 Memory Barrier。
```

```java
public class SingletonDemo {
    private static volatile SingletonDemo instance; // volatile 防止重排序

    private SingletonDemo() {
        System.out.println(Thread.currentThread().getName() + "\t get instance");
    }
    
    public static SingletonDemo getInstance() {
        if (instance == null) {
            synchronized (SingletonDemo.class) {
                if (instance == null)
                    instance = new SingletonDemo();
            }
        }
        return instance;
    }

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            new Thread(() -> {
                getInstance();
            }, String.valueOf(i)).start();
        }
    }
}
```

```shell
# 单例模式 DCL<Double Check Lock 双端检锁机制> 代码

# instance = new SingletonDemo() 是被分成以下 3 步完成
	# memory = allocate();     分配对象内存空间
	# instance(memory);        初始化对象
	# instance = memory;	   设置 instance 指向刚分配的内存地址，此时 instance != null
	
	# 步骤2 和 步骤3 不存在数据依赖关系，重排与否的执行结果单线程中是一样的
	# 这种指令重排是被 Java 允许的
	# 当 3 在前时，instance 不为 null，但实际上初始化工作还没完成，
		# 会变成一个返回 null 的getInstance
```

```shell
# volatile 只能用于修饰成员变量，如果业务代码中有较多的局部变量需要禁止 指令重排，
	# 可以使用 JDK1.7 之后出现的 Unsafe 类中 API 手动添加 内存屏障。
	
	# 一般不建议使用 Unsafe 类，容易造成内存泄漏。
	
	# Unsafe 类是由 启动类加载器(BootStrapClassLoad) 加载的，不能直接 new 出来
```

```java
/**
 * 通过反射获取 Unsafe 类
 */
public class UnsafeUtil {
  
  public static Unsafe getUnsafe() {
    
     try {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        return (Unsafe) field.get(null);
      } catch {
        e.printStackTrace();
      }
      return null;
  }
  
}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603078034124.png)

### volatile 内存语义的实现

```shell
# 为了实现 volatile 内存语义，
	# JMM 会分别限制 编译器重排序 和 处理器重排序。
```

#### JMM 针对编译器制定的 volatile 重排规则表

| 第一个操作 | 第二个操作: 普通读写 | 第二个操作: volatile读 | 第二个操作: volatile 写 |
| ---------- | -------------------- | ---------------------- | ----------------------- |
| 普通读写   | 可以重排             | 可以重排               | 不可以重排              |
| volatile读 | 不可以重排           | 不可以重排             | 不可以重排              |
| volatile写 | 可以重排             | 不可以重排             | 不可以重排              |

#### Java的实现

```shell
# 编译器在生成字节码时，会在 指令序列 中插入 内存屏障，
	# 来禁止 特定类型 的处理器重排序。
	
# 在每个 volatile 写操作的前面，插入一个 StoreStore 屏障

# 在每个 volatile 写操作的后面，插入一个 StoreLoad 屏障

# 在每个 volatile 读操作的前面，插入一个 LoadLoad 屏障

# 在每个 volatile 写操作的后面，插入一个 LoadStore 屏障
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603079163346.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603079257073.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603079274257.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1603079289793.png)

