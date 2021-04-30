[TOC]

# 周阳 - Java 底层<上>

## JUC 多线程及高并发

### 请谈谈你对 volatile 的理解

```shell
# volatile 是 Java 虚拟机提供的轻量级的同步机制。
	# 保证可见性
	# 不保证原子性
		# A 线程写操作并同步到主存后，通知到其他线程这个时间差上，别的线程也同步写操作到主存上
		# 这就是没有保证原子性，出现了写覆盖
		# 如何解决原子性问题？
			# 加 sycnchronized
			# 用 AtomicInteger<推荐>
	# 禁止指令重排
		# 利用内存屏障<Memory Barrier，一个 CPU 指令>,它的两个作用
        	# 保证特定操作的执行顺序
        	# 保证某些变量的内存可见性<即 volatile 保证可见性的来源>
```

#### volatile 可见性代码验证

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

#### volatile 不保证原子性代码验证

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
# javap -c 配置 https://jingyan.baidu.com/article/f71d6037c05ecc1ab741d163.html
# 利用 javap -c 命令编译得出字节码 i++ 实际指令数及步骤为 : 
```

![UTOOLS1569217383589.png](https://i.loli.net/2019/09/23/jWMfVpSErCaiPg3.png)

#### 请谈谈你对 JMM 的理解

```shell
# JMM<Java Memory model | Java 内存模型>
	# 抽象概念，并不真实存在。
	# 它描述一组规则或规范。
	# 通过这组规范定义了程序中各个变量(包括实例字段、静态字段和构成数组对象的元素)的访问方式
	
# JMM 关于同步的规定
	# 线程解锁前，必须把共享变量的值刷新回主内存。
	# 线程加锁前，必须读取主内存的最新值到自己的工作内存。
	# 加锁解锁是同一把锁。
	
# 可见性 : A B 线程各自拷贝主存变量并操作，先写回主存的值一定要通知给另外的线程
    # 而 JMM 中线程可见性的实现是由 : 
        # Volatile
        # Synchronized
        # final 来实现的
        
# 原子性 : 即某个线程正在做某个具体业务时，中间不可以被加塞或者被分隔
	# 要么同时成功，要么同时失败
	
# 有序性 : 计算机在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排
	# 一般分为以下三种
		# 源代码 -> [编译器优化的重排] -> [指令并行的重排] -> [内存系统的重排] -> 最终执行指令
	# 单线程环境确保程序最终执行结果和代码顺序执行的结果一致
	# 处理器在进行重排序时必须要考虑指令之间的 数据依赖性
	# 多线程环境中线程交替执行，此时编译器优化重排就可能导致，多个线程使用的同一个变量无法保证一致性。
	
# 由于 JVM 运行程序的实体是线程，而每个线程创建时 JVM 都会为其创建一个工作内存<线程栈>。
	# 工作内存是每个线程的私有数据区域。
	# 而 JMM 中规定所有变量都存储在 主内存 中，即所有线程都可以访问。
	# 但线程对变量的操作(读写)必须在工作内存中进行。
	# 流程
        # 线程栈本地内存
        # -> 从主内存中拷贝变量
        # -> 对本地内存中的变量进行读写操作
        # —> 操作完成后再将其写会主内存
    # 因此，不同的线程间无法直接互访，线程间的通信必须通过主内存来完成。
```

![UTOOLS1569205680742.png](https://i.loli.net/2019/09/23/3CIPLJovnMZsQl1.png)

#### 你在哪些地方用到过 volatile

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

### 请谈谈你对 CAS 的理解

```shell
# CAS 是什么？
	# 答: 比较并交换<compareAndSet>,它是一条 CPU 并发原语
		# 它的功能是判断内存某个位置的值是否为预期值，如果是则更改为新的值，这个过程是原子性的。
	
	# 例: AtomicInteger 的 compareAndSet('期望值','设置值') 方法
		# 期望值与目标值一致时，修改目标变量为设置值
		# 期望值与目标值不一致时，返回 false 和最新主存的变量值
		
# CAS 的底层原理
	# 例: AtomicInteger.getAndIncrement()
	# 调用 Unsafe 类中的 CAS 方法，JVM 会帮我们实现出 CAS 汇编指令
		# 这是一种完全依赖于硬件的功能，通过它实现原子操作。
		# 原语的执行必须是连续的，在执行过程中不允许被中断，CAS 是 CUP 的一条原子指令。
```

#### AtomicInteger.getAndIncrement() 方法详解

![UTOOLS1569231184344.png](https://i.loli.net/2019/09/23/fym1bIkcrhOUgl7.png)

```java
/**
 * unsafe: rt.jar/sun/misc/Unsafe.class
 *   Unsafe 是 CAS 的核心类，由于 Java 无法直接访问底层系统，需要通过本地<native>方法来访问
 *	 Unsafe 相当于一个后门，基于该类可以直接操作特定内存的数据
 *	 Unsafe 其内部方法都是 native 修饰的，可以像 C 的指针一样直接操作内存
 *	 Java 中的 CAS 操作执行依赖于 Unsafe 的方法，直接调用操作系统底层资源执行程序
 *
 * this: 当前对象
 *	 变量 value 由 volatile 修饰，保证了多线程之间的内存可见性、禁止重排序
 *
 * valueOffset: 内存地址
 *	 表示该变量值在内存中的偏移地址，因为 Unsafe 就是根据内存偏移地址获取数据
 *
 * 1: 固定写死，原值加1
 */
public final int getAndIncrement(){
    return unsafe.getAndAddInt(this,valueOffset,1);
}

/**
 * Unsafe.getAndAddInt()
 * getIntVolatile: 通过内存地址去主存中取对应数据
 * 
 * while(!this.compareAndSwapInt(var1,var2,var5,var5 + var4)):
 * 	 将本地 value 与主存中取出的数据对比，如果相同，对其作运算，
 * 		此时返回 true，取反后 while 结束，返回最终值。
 * 	 如果不相同，此时返回 false，取反后 while 循环继续运行，此时为自旋锁<重复尝试>
 *		由于 value 是被 volatile 修饰的，所以拿到主存中最新值，再循环直至成功。
 */
public final int getAndAddInt(Object var1,long var2,int var4){
    int var5;
    do{
        var5 = this.getIntVolatile(var1,var2); // 从主存中拷贝变量到本地内存
    } while(!this.compareAndSwapInt(var1,var2,var5,var5 + var4));
    return var5;
}
```

#### CAS 代码演示

```java
public class CASDemo {

    public static void main(String[] args) {
        AtomicInteger num = new AtomicInteger(5);
        // TODO
        System.out.println(num.compareAndSet(5, 1024) + "\t current num" + num.get());
        System.out.println(num.compareAndSet(5, 2019) + "\t current num" + num.get());
    }
```

#### CAS 缺点

```shell
# 如果 CAS 长时间一直不成功，会给 CPU 带来很大的开销

# 只能保证一个共享变量的原子操作

# 引出了 ABA 问题
```

### 请你谈谈原子类的ABA问题、原子更新引用问题

#### ABA 问题

```shell
# CAS 会导致 ABA 问题

# CAS 算法实现一个重要前提
	# 去除内存中某时刻的数据，并在当下时刻比较并替换
	# 那么这个时间差内会导致数据的变化
	
# 例: A、B线程从主存取出变量 value
	# -> A 在 N次计算中改变 value 的值
	# -> A 最终计算结果还原 value 最初的值
	# -> B 计算后，比较主存值与自身 value 值一致，修改成功
	
# 尽管各个线程的 CAS 都操作成功，但是并不代表这个过程就是没有问题的
```

#### ABA问题代码验证

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class ABADemo{

    static AtomicReference<Integer> atomicReference = new AtomicReference<>(100);

    public static void main(String[] args) {
        new Thread(() -> {
            atomicReference.compareAndSet(100,101);
            atomicReference.compareAndSet(101,100);
            },"AtomicReference Thread 1").start();

        new Thread(() -> {
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            System.out.println(atomicReference.compareAndSet(100, 101) + "\t" + atomicReference.get());
        },"AtomicReference Thread 2").start();
    }

}
```

#### 原子引用问题

```shell
# java.util.concurrent.atomic.AtomicReference
	# Class AtomicReference<V> : 原子引用，V 为需要包装的泛型
```

##### 原子引用验证代码

```java
@Getter
@ToString
@AllArgsConstructor
class User{
    String username;
    int age;
}

public class AtomicReferenceDemo {

    public static void main(String[] args) {
        User tony = new User("tony",20);
        User tom = new User("tom",25);

        AtomicReference<User> reference = new AtomicReference<>();
        reference.set(tony);
        System.out.println(reference.compareAndSet(tony, tom) + "\t user : " + reference.get().toString());
        System.out.println(reference.compareAndSet(tony, tom) + "\t user : " + reference.get().toString());

    }

}
```

#### 时间戳原子引用解决 ABA 问题

```shell 
# java.util.concurrent.atomic.AtomicStampedReference
	# Class AtomicStampedReference<V> : 时间戳原子引用，V 为需要包装的泛型
```

#### 时间戳原子引用解决 ABA 问题代码验证

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicStampedReference;

public class ResolveABADemo {

    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(100, 1);

    public static void main(String[] args) {
        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "初始版本号" + stamp);
            try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }
            atomicStampedReference.compareAndSet(100, 101, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            atomicStampedReference.compareAndSet(101, 100, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1);
            System.out.println(Thread.currentThread().getName() + "结束版本号" + atomicStampedReference.getStamp());
        }, "AtomicStampedReference Thread 1").start();

        new Thread(() -> {
            int stamp = atomicStampedReference.getStamp();
            System.out.println(Thread.currentThread().getName() + "初始版本号" + stamp);
            try { TimeUnit.SECONDS.sleep(3); } catch (InterruptedException e) { e.printStackTrace(); }
            boolean result = atomicStampedReference.compareAndSet(100, 101, stamp, stamp + 1);
            System.out.println(Thread.currentThread().getName() + "结束版本号" + atomicStampedReference.getStamp() + "\t CAS 操作结果 : " + result);
            System.out.println("最终主存值为 : " + atomicStampedReference.getReference() + "\t 最终版本号为 : " + atomicStampedReference.getStamp());
        }, "AtomicStampedReference Thread 2").start();
    }

}
```

### 请编码一个线程不安全的ArrayList 案例并给出解决方案

#### ArrayList 线程不安全代码验证

```java
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * java.util.ConcurrentModificationException : 并发修改异常
 */
public class ContainerNotSafeDemo {

    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        for(int i = 1;i <= 30; i++ )
        {
            new Thread(() -> {
                list.add(UUID.randomUUID().toString().substring(0,8));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }

}
```

#### 导致原因

```shell
# 并发争抢修改导致
```

#### 解决方案

```shell
# Vector
	# JDK1.0 推出 List 的实现类，add 方法由 synchronized 修饰
	# 基本不会有人用，因为 synchronized 同步代码块对并发性能影响太大
	
# Collections.synchronizedList(new ArrayList<>());
	
# CopyOnWriteArrayList <最终方案>
	# 写时复制
		# 原理即下述源码
		# 支持并发读，而不需要加锁
		# 读写分离思想，读和写是用的不同容器
	# Object[] element : 元素数组是由 volatile 修饰
	# add() 方法中追加 ReetrantLock 锁
	# 源码如下 :
```

```java
public boolean add(E e){
    final ReetrantLock lock = this.lock;
    lock.lock();
    try{
        Object[] elements = getArray();
        int len = elements.length;
    	Object[] newElements = Arrays.copyOf(elements,len+1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

### 请给出一个线程不安全的 HashSet案例并给出解决方案

#### HashSet 线程不安全代码验证

```java
import java.util.HashSet;
import java.util.List;
import java.util.Set;
import java.util.UUID;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * java.util.ConcurrentModificationException : 并发修改异常
 */
public class ContainerNotSafeDemo {

    public static void main(String[] args) {
        Set<String> set = new HashSet<>();
        for(int i = 1;i <= 30; i++ )
        {
            new Thread(() -> {
                set.add(UUID.randomUUID().toString().substring(0,8));
                System.out.println(set);
            },String.valueOf(i)).start();
        }

    }
}
```

#### 解决方案

```shell
# Collections.synchronizedSet(new HaseSet<>());

# CopyOnWriteArraySet
	# 构造方法中举着 Set 的牌子，其实用的是 CopyOnWriteArrayList
		# 在 add() 方法中扩展了 indexOf() 方法遍历数组，看add元素是否已存在
	# 扩展 : HashSet 的构造方法其实是创建了一个容量为 16，负载因子为 0.75 的 HashMap
		# HashSet KV 中元素值为 K,V 是一个常量
```

### 请给出一个线程不安全的 HashMap案例并给出解决方案

#### HashMap 线程不安全代码验证

```java
import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

/**
 * java.util.ConcurrentModificationException : 并发修改异常
 */
public class ContainerNotSafeDemo {

    public static void main(String[] args) {
        Map<String,Object> map = new HashMap<>();
        for(int i = 1;i <= 30; i++ )
        {
            new Thread(() -> {
                map.put(Thread.currentThread().getName(),UUID.randomUUID().toString().substring(0,8));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
        
    }
}
```

#### 解决方案

```shell
# Collections.synchronizedMap(new HashMap<>())

# ConcurrentHashMap
```

### 请你谈谈对Java 锁的理解

#### Sychronized 与 Lock

```shell
# 1. 原始构成
    # sychronized 是关键字，属于 JVM 层面
        # 底层依赖的是汇编语言中 monitor 对象
            # monitorenter : 进入锁
            # monitorexit : 离开锁
        # wait/notify 等方法也是依赖 monitor对象
            # 只有在同步代码块或方法中才能调用 wait/notify 等方法
       
    # Lock 是 java.util.concurrent.locks.Lock，属于 Api 层面的锁
    
# 2. 使用方法
	# sychronized 不需要用户手动释放锁，当同步代码执行完后，系统会自动让线程释放对锁的占用
		# sychronized 中汇编会释放两次锁，一次正常退出，一次异常退出，所以不会出现死锁的情况<抛异常>

	# ReentrantLock 则需要用户手动释放锁，否则会出现死锁的情况
	
# 3. 等待是否可中断
	# synchronized 不可中断，除非抛出异常或者正常运行完成
	
	# ReentrantLock 可中断
		# 1. 设置超时方法，tryLock(long timeout,TimeUnit unit)
		# 2. lockInterruptibly() 放代码块中，调用 interrupt() 方法可中断
		
# 4. 加锁是否公平
	# synchronized 是非公平锁
	
	# ReentrantLock 两者都可以，默认非公平
		# 构造方法传入 true 时为公平锁
		# 源码中是依赖 AbstartQueueSynchronized<CompareAndSetStatu> 实现公平
		
# 5. 锁绑定多个条件 Condition
	# synchronized 没有
	# ReentrantLock 用来实现分组唤醒需要唤醒的线程们
		# 可以精确唤醒，而不像 synchronized 要么随机唤醒一个线程，要么唤醒全部线程
```

##### Lock Condition 分段锁代码验证

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockConditionDemo {

    public static void main(String[] args){
        ShareData shareData = new ShareData();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                shareData.print15();
            }
        },"C").start();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                shareData.print5();
            }
        },"A").start();

        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                shareData.print10();
            }
        },"B").start();

    }

}

class ShareData{
    private int number = 0;
    private Lock lock = new ReentrantLock();
    private Condition A = lock.newCondition();
    private Condition B = lock.newCondition();
    private Condition C = lock.newCondition();

    public void print5(){

        lock.lock();
        try {
            // 1. 判断
            while (number != 0){
                // 等待
                A.await();
            }
            // 2. 干活
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + number);
            }
            number ++;
            // 3. 通知
            B.signal();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print10(){

        lock.lock();
        try {
            // 1. 判断
            while (number != 1){
                // 等待
                B.await();
            }
            // 2. 干活
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + number);
            }
            number ++;
            // 3. 通知
            C.signal();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void print15(){

        lock.lock();
        try {
            // 1. 判断
            while (number != 2){
                // 等待
                C.await();
            }
            // 2. 干活
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + "\t" + number);
            }
            number = 0;
            // 3. 通知
            A.signal();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

#### 公平和非公平锁

```shell
# 公平锁
	# 指多个线程按照申请锁的顺序来获取锁 <队列，先入先出>
	
# 非公平锁
	# 指多个线程获取锁的顺序并不是按照申请锁的顺序
	# 高并发情况下，可能造成优先级反转或者饥饿现象
	# 吞吐量比公平锁大
	
# 注意
	# Synchronized 也是一种非公平锁
```

##### 公平和非公平锁代码示例

```java
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class LockDemo {

    public static void main(String[] args) {
        Lock noFailLock = new ReentrantLock();
        Lock failLock = new ReentrantLock(true);
    }

}
```

#### 可重入锁<又名递归锁>

```shell
# 指同一线程外层函数获得锁之后，内层递归函数仍然能够获取该锁的代码
	# 在同一个线程，外层方法获取锁的时候，进入内层方法会自动获取锁
	# 即，线程可以进入任何一个它已经拥有的锁所同步着的代码块
	
# ReentrantLock 和 Sychronized 就是典型的可重入锁

# 可重入锁最大的作用就是避免死锁

# lock 和 unlock,无论几把锁，只要配对即可正确运行程序
```

##### 可重入锁代码验证

```java
public class ReentrantLockDemo {

    public static void main(String[] args){
        Phone phone = new Phone();
        new Thread(() -> {
            try {
                phone.sendSms();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"A").start();

        new Thread(() -> {
            try {
                phone.sendSms();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"B").start();
    }

}

class Phone{

    public synchronized void sendSms() throws Exception{
        System.out.println(Thread.currentThread().getName() + "\t invoked sendSms()");
        sendEmail();
    }

    public synchronized void sendEmail() throws Exception{
        System.out.println(Thread.currentThread().getName() + "\t invoked sendEmail()");
    }

}
```

#### 自旋锁<spinlock>

```shell
# 指尝试获取锁的线程不会立即阻塞，而是采用循环的方式去尝试获取锁
	# 好处 : 减少线程上下文切换的消耗
	# 缺点 : 循环会消耗 CPU
```

##### 自旋锁代码验证

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

public class SpinLockDemo {

    // 原子引用线程
    AtomicReference<Thread> atomicReference = new AtomicReference<>();

    public static void main(String[] args){
        SpinLockDemo spinLockDemo = new SpinLockDemo();
        new Thread(() -> {
            spinLockDemo.mylock();
            try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
            spinLockDemo.myUnlock();
        },"A").start();

        try { TimeUnit.SECONDS.sleep(1); } catch (InterruptedException e) { e.printStackTrace(); }

        new Thread(() -> {
            spinLockDemo.mylock();
            spinLockDemo.myUnlock();
        },"B").start();
    }

    public void mylock(){
        Thread thread = Thread.currentThread();
        System.out.println(Thread.currentThread().getName() + "\t come in ^_^");
        while (!atomicReference.compareAndSet(null,thread)){}
    }

    public void myUnlock(){
        Thread thread = Thread.currentThread();
        atomicReference.compareAndSet(thread,null);
        System.out.println(Thread.currentThread().getName() + "\t go out ^_^");
    }

}
```

#### 读写锁<独占锁(写锁)/共享锁(读锁)/互斥锁>

```shell
# 独占锁
	# 指该锁一次只能被一个线程所持有
	# 对 ReentrantLock 和 Synchronized 而言都是独占锁
	
# 共享锁
	# 指该锁可被多个线程所持有
	# 对 ReentrantReadWriteLock 其读锁是共享锁，其写锁是独占锁
	# 读锁的共享锁可保证并发读是非常高效的，读写，写读，写写的过程是互斥的
```

##### 读写锁代码验证

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockDemo {

    public static void main(String[] args) {
        MyCache myCache = new MyCache();
        for (int i = 1; i <= 50; i++) {
            final int tempInt = i;
            new Thread(() -> {
                try {
                    myCache.put(tempInt + "", tempInt);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }

        for (int i = 1; i <= 50; i++) {
            final int tempInt = i;
            new Thread(() -> {
                try {
                    myCache.get(tempInt + "");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }, String.valueOf(i)).start();
        }

    }
}

class MyCache {
    private volatile Map<String, Object> map = new HashMap<>();
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    /**
     * 写操作 : 原子 + 独占
     */
    public void put(String key, Object value) {

        lock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在写入 : " + key);
            TimeUnit.MILLISECONDS.sleep(300);
            map.put(key, value);
            System.out.println(Thread.currentThread().getName() + "\t 写入完成 ! ");
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.writeLock().unlock();
        }

    }

    public void get(String key) {

        lock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 正在读取 : " + key);
            TimeUnit.MILLISECONDS.sleep(300);
            Object result = map.get(key);
            System.out.println(Thread.currentThread().getName() + "\t 读取完成 : " + result);
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.readLock().unlock();
        }

    }
}
```

### CountDownLatch

```shell
# CountDownLatch
	# 可以理解为火箭发射
	# main 线程为需要发射的火箭，其余线程所作操作为火箭发射的倒计时数
	# 只有其余线程操作完毕后，火箭才能发射
	
# 让一些线程阻塞直至另一些线程完成一系列操作后才被唤醒
	# CountDownLatch 主要有两个方法，当一个或多个线程调用 await 方法时，调用线程会被阻塞
	# 其他线程调用 countDown 方法会将计数器减一
	# 当计数器的值变为0时，因调用 await 方法被阻塞的线程会被唤醒，继续执行
```

#### CountDownLatch 代码验证

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {

    public static void main(String[] args) {

        CountDownLatch countDownLatch = new CountDownLatch(6);

        for(int i = 1;i <= 100; i++ )
        {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 上完自习，离开教室");
                countDownLatch.countDown();
            },String.valueOf(i)).start();
        }

        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "\t 班长最后关门走人");

    }

}
```

### CyclicBarrier

```shell
# CyclicBarrier
	# 可以理解为集齐七颗龙珠，召唤神龙
	
# 可循环<Cyclic>使用的屏障<Barrier>
	# 让一组线程到达一个屏障时，被阻塞
	# 直到最后一个线程到达屏障时，屏障才会开门
	# 所有被屏障拦截的线程才会继续干活
	# 线程进入屏障通过 CyclicBarrier 的 await() 方法
```

#### CyclicBarrier 代码验证

```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,() ->
            System.out.println("哈撒给 ！\t 召唤神龙")
        );
        
        for(int i = 1;i <= 7; i++ )
        {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + "\t 收集到龙珠");
                try { cyclicBarrier.await(); } catch (Exception e) { e.printStackTrace(); }
            },String.valueOf(i)).start();
        }
    }

}
```

### Semaphore

```shell
# 可以理解为停车场
	# 例 : 100 辆车抢 10 个停车位
	
# 两个目的
	# 一 : 用于多个共享资源的互斥使用
	# 二 : 用于并发线程数的控制
```

#### Semaphore 代码验证

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {

    public static void main(String[] args) {
        // 模拟 10 个停车位
        Semaphore semaphore = new Semaphore(10);

        // 模拟 100 辆车
        for(int i = 1;i <= 100; i++ )
        {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + "\t 抢到车位");
                    TimeUnit.SECONDS.sleep(3);
                } catch (Exception e){
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                    System.out.println(Thread.currentThread().getName() + "\t 离开车位");
                }

            },String.valueOf(i)).start();
        }

    }

}
```

### 请谈谈对阻塞队列的理解

```shell
# 阻塞队列
	# 阻塞队列为空时，从队列中获取元素的操作将会被阻塞
	# 阻塞队列为满时，往队列里添加元素的操作将会被阻塞
	
# 阻塞队列的好处
	# 多线程领域中，所谓阻塞，即某些情况下会挂起线程，一旦条件满足，线程唤醒。
	
# 为什么需要 BlockingQueue
	# 我们不需要关心什么时候需要阻塞线程，什么时候需要唤醒线程了
	# 在 JUC 包发布以前，多线程环境下，程序员需要自己控制这些细节，并且兼顾效率与线程安全
```

#### 种类分析

```shell
# ArrayBlockingQueue
	# 数组结构组成的有界阻塞队列
	
# LinkedBlockingQueue
	# 由链表结构组成的有界(但大小默认值为 Integer.MAX_VALUE) 阻塞队列
	
# PriorityBlockingQueue
	# 支持优先级排序的无界阻塞队列
	
# DelayQueue
	# 使用优先级队列实现的延迟无界阻塞队列
	
# SynchronousQueue
	# 不存储元素的阻塞队列，也即单个元素的队列
	
# LinkedTransferQueue
	# 由链表结构组成的无界阻塞队列
	
# LinkedBlockingDeque
	# 由链表结构组成的双向阻塞队列
```

#### 核心方法

```shell
# 抛出异常组
	# add(e)
		# 队列满时 add 会抛出 java.lang.IllegalStateException: Queue full
	# remove()
		# 队列空时 remove 会抛出 java.util.NoSuchElementException
	# element()
		# 得到队首元素，队列为空时，抛出 java.util.NoSuchElementException
		
# 返回布尔值组
	# offer(e)
		# 往阻塞队列插入数据，成功时返回 true，失败时返回 false
	# poll()
		# 从阻塞队列取出数据，成功时返回 数据，队列为空时返回 null
	# peek()
		# 取出队首元素，成功时返回 数据，队列为空时返回 null
		
# 阻塞
	# put(e)
		# 往阻塞队列插入数据，无返回值，插入不成功时阻塞线程，直至插入成功 Or 线程中断
	# take()
		# 从阻塞队列取出数据，成功返回数据，不成功时阻塞线程，直至取出成功 Or 线程中断

# 超时
	# offer(e,time,unit)
		# 往阻塞队列插入数据，成功返回 true，不成功时线程阻塞等待超时时间，过时返回false 并放弃操作
	# poll(time,unit)
		# 从阻塞队列取出数据，成功返回 数据，队列为空时线程阻塞等待超时时间，过时返回false 并放弃操作
```

#### SynchronousQueue 代码验证

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.TimeUnit;

public class SynchronousQueueDemo {

    public static void main(String[] args){

        BlockingQueue<String> blockingQueue = new SynchronousQueue<>();
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "\t put A");
                blockingQueue.put("A");
                System.out.println(Thread.currentThread().getName() + "\t put A");
                blockingQueue.put("A");
                System.out.println(Thread.currentThread().getName() + "\t put A");
                blockingQueue.put("A");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"A").start();

        new Thread(() -> {

            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println(Thread.currentThread().getName() + "\t take A");
                blockingQueue.take();
                System.out.println(Thread.currentThread().getName() + "\t take A");
                blockingQueue.take();
                System.out.println(Thread.currentThread().getName() + "\t take A");
                blockingQueue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        },"B").start();

    }

}
```

#### 阻塞队列的使用场景

```shell
# 生产者消费者模式
	
# 线程池

# 消息中间件
```

##### 传统版生产者消费者模式 Demo

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * @Author DuChao
 * @Date 2019-10-11 17:00
 * 题目 : 一个初始值为0的变量，两个线程对其进行操作，一个+1 一个-1，交替五轮
 *
 * 1. 线程    操作      资源类
 * 2. 判断    干活      通知
 * 3. 防止虚假唤醒机制
 */
public class ProdCons_TraditionDemo {

    public static void main(String[] args){
        ShareData shareData = new ShareData();
        new Thread(() -> {
            for (int i = 0; i <=5; i++) {
                try {
                    shareData.increment();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
            },"A").start();

        new Thread(() -> {
            for (int i = 0; i <=5; i++) {
                try {
                    shareData.decrement();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        },"B").start();
    }

}

class ShareData{
    private int number = 0;
    private Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();

    public void increment() throws Exception{

        lock.lock();
        try {
            // 1. 判断
            while (number != 0){
                // 等待，不能生产
                condition.await();
            }
            // 2. 干活
            number ++;
            System.out.println(Thread.currentThread().getName() + "\t" + number);

            // 3. 通知
            condition.signalAll();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void decrement() throws Exception{

        lock.lock();
        try {
            // 1. 判断
            while (number == 0){
                // 等待，不能生产
                condition.await();
            }
            // 2. 干活
            number --;
            System.out.println(Thread.currentThread().getName() + "\t" + number);

            // 3. 通知
            condition.signalAll();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

##### 阻塞队列版生产者消费者模式Demo

```java
import cn.hutool.core.util.StrUtil;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class ProdCons_BlockingQueueDemo {

    public static void main(String[] args) {
        ShareData shareData = new ShareData(new ArrayBlockingQueue<>(10));
        new Thread(() -> {
            try {
                shareData.prod();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"prod").start();

        new Thread(() -> {
            try {
                shareData.cons();
            } catch (Exception e) {
                e.printStackTrace();
            }
        },"cons").start();

        try {
            TimeUnit.SECONDS.sleep(5);
            shareData.stop();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}

class ShareData {

    private volatile boolean flag = true;
    private AtomicInteger num = new AtomicInteger();

    BlockingQueue<String> queue = null;

    public ShareData(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    public void prod() throws Exception {
        String data = null;
        boolean value;
        while (flag) {
            data = num.incrementAndGet() + "";
            value = queue.offer(data, 2, TimeUnit.SECONDS);
            if (value)
                System.out.println(Thread.currentThread().getName() + "\t 插入队列 :" + data + " 成功");
            else
                System.out.println(Thread.currentThread().getName() + "\t 插入队列 :" + data + " 失败");
            TimeUnit.SECONDS.sleep(1);
        }
        System.out.println(Thread.currentThread().getName() + "\t 停止生产");
    }

    public void cons() throws Exception {
        String result = null;
        while (flag) {
            result = queue.poll(2,TimeUnit.SECONDS);
            if (StrUtil.isBlank(result)) {
                System.out.println(Thread.currentThread().getName() + "\t 消费队列为空,超时停止");
                flag = false;
                return;
            }
            System.out.println(Thread.currentThread().getName() + "\t 消费队列 :" + result + " 成功");
        }

    }

    public void stop(){
        flag = false;
    }
}
```

### 请谈谈对Callable 接口的理解

```shell
# 获取线程的方式
	# 继承 Thread 类
	# 实现 Runnable 接口
	# 实现 Callable 接口
	# 使用线程池
	
# FetureTask<> 实现 callable、Runnabel 接口，获取一个异步计算并带返回值的线程
```

#### FetureTask 代码验证

```java
import java.util.concurrent.FutureTask;

public class CallableDemo {

    public static void main(String[] args) throws Exception{
        FutureTask<String> task = new FutureTask<>(() -> {
            System.out.println(Thread.currentThread().getName() + "\t FutureTask come in");
            return "快乐的一只小青蛙";
        });

        new Thread(task,"Future").start();
        System.out.println(task.get());
    }

}
```

### 为什么用线程池？优势在哪？

```shell
# 线程池主要控制运行线程的数量
	# 处理过程中，将任务放入队列
	# 线程创建后启动这些任务
	# 如果线程数量超过了最大数量限制，则排队等候
	# 等待其它线程执行完毕，再从队列中取出任务执行
	
# 线程池其实就是阻塞队列

# 主要特点
	# 线程复用
	# 控制最大并发数
	# 管理线程
	
# Java 中的线程池是通过 Executor 框架实现的
	# 该框架中用到了 Executor、Executors、ExecutorService、ThreadPoolExecutor 这几个类
```

![UTOOLS1571032812959.png](https://i.loli.net/2019/10/14/CDzUX7oprfdVlbA.png)

### 线程池3个常用方式

```shell
# JDK8 已经有五种线程使用方式

# 重点常用的三种
	# Executors.newFixedThreadPool(int)
		# 固定队列容量线程池
		# BlokingQueue 是 LinkedBlokingQueue()，有界队列，但是容量为 Integer.MAX_VALUE
	
	# Executors.newSingleThreadExecutor()
		# 单线程容量线程池
		# 一样也是 LinkedBlokingQueue()
	
	# Executors.newCacheThreadPool()
		# N个线程容量线程池
		# 由并发量决定池子中的线程数
		# SynchronousQueue
```

#### FixedThreadPool 代码验证

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolDemo {

    public static void main(String[] args){
        // 1池5个处理线程
        ExecutorService pool = Executors.newFixedThreadPool(5);

        try {
            for (int i = 1; i <= 10; i++) {
                pool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 办理业务!");
                });
            }
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            pool.shutdown();
        }
    }

}
```

### 线程池七大参数入门简介

```shell
# 一个银行网点 <线程池>
	# -> 共 10* 个窗口 <maximumPoolSize 最大线程数>
	# -> 开放 5* 个窗口 <corePoolSize 核心线程数>
	# -> 今天办理业务的特别多，其余5个窗口加班一天* <keepAliveTime + unit 多余线程存活时间+单位>
	# -> 办理业务的人在窗口前排队* <workQueue 请求任务的阻塞队列>
	# -> 银行*里的A*职员、B*职员... 给办理业务 <threadFactory 产生线程、线程名、线程序数...>
	# -> 最多排10个，来了11个，并且每个窗口都有人在办理业务，多的人怎么拒绝*呢？<handler 拒绝策略>

# corePoolSize
	# 线程池中的常驻核心线程数
	# 创建线程池后，当有请求任务进来，就安排池中的线程去执行请求任务
	# 当线程池中的线程数目达到 corePoolSize 后，就会把到达的任务放到缓存队列中
	
# maximumPoolSize
	# 线程池能够容纳同时执行的最大线程数，此值必须大于等于1
	
# keepAliveTime
	# 多余的空闲线程的存活时间
	# 当前线程池数量超过 corePoolSize 时，当空闲时间达到 keepAliveTime 值时，
		# 多余空闲线程会被销毁直到只剩下 corePoolSize 个线程为止
		
# unit
	# keepAliveTime 的单位
	
# workQueue
	# 任务队列，被提交但尚未被执行的任务
	
# threadFactory
	# 表示生成线程池中工作线程的线程工厂<线程名字、线程序数...>
	# 用于创建线程一般用默认的即可
	
# handler
	# 拒接策略，表示当队列满了并且工作线程大于等于线程池的最大线程数(maximumPoolSize)时
		# 如何拒绝新的任务
```

### 请你谈谈线程池的底层工作原理？

![UTOOLS1571104813186.png](https://i.loli.net/2019/10/15/aHzDS4Od9uRMk6V.png)

```shell
# 1. 创建线程池后，等待请求任务

# 2. 当调用 execute() 方法添加请求任务时，线程池做如下判断
	# 如果正在运行的线程数量小于 corePoolSize，马上创建线程执行请求任务
	# 如果正在运行的线程数量大于或等于 corePoolSize，将请求任务放入阻塞队列
	# 如果阻塞队列满了，且正在运行的线程数小于 mamimumPoolSize,创建非核心线程执行请求任务
	# 如果队列满了且线程池线程达到最大线程数，线程池启动饱和拒绝策略来执行
	
# 3. 当一个线程完成任务时，从阻塞队列中取出下一个任务来执行

# 4. 当一个线程无事可做超过一定时间<keepAliveTime>时，线程池会判断
	# 如果当前运行的线程数大于 corePoolSize，该线程被销毁
	# 所以，线程池完成所有请求任务后，最终会收缩到 corePoolSize 的大小
```

### 线程池的4种拒绝策略理论简介

```shell
# JDK 内置的拒绝策略
	# AbortPolicy(默认)
		# 直接抛出 RejectedExecutionException 异常阻止系统正常运行
		
	# CallerRunsPolicy
		# "调用者运行" 一种调节机制
		# 该策略既不会抛弃任务，也不会抛出异常
		# 而是将某些任务回退到调用者，从而降低新任务的流量
		
	# DiscardOldestPolicy
		# 抛弃队列中等待最久的任务
		# 然后把当前任务中加入队列中尝试再次提交当前任务
		
	# DiscardPolicy
		# 直接丢弃任务，不予任何处理也不抛出异常
		# 如果允许任务丢失，这是最好的一种方案
		
# 以上拒绝策略都是实现了 RejectedExecutionHandler 接口
```

### 线程池在实际生产中使用哪一个

```shell
# 阿里巴巴 Java 开发手册
	# 线程池不允许使用 Executors 创建，而是通过 ThreadPoolExecutor 的方式
	
	# FixedThreadPool 和 SingleThreadPool
		# 允许的阻塞队列容量为 Integer.MAX_VALUE，可能会堆积大量的请求，导致 OOM
		
	# CachedThreadPool 和 ScheduledThreadPool
		# 允许的创建线程数量为 Integer.MAX_VALUE,可能会创建大量的线程，导致 OOM
```

### 线程池的手写改造和拒绝策略

```java
import java.util.concurrent.*;

public class ThreadPoolDemo {

    public static void main(String[] args) {
        ExecutorService threadPool =
                new ThreadPoolExecutor(2,
                        5,
                        1L,
                        TimeUnit.SECONDS,
                        new LinkedBlockingDeque<Runnable>(3),
                        Executors.defaultThreadFactory(),
                        new ThreadPoolExecutor.CallerRunsPolicy());

            for(int i = 1;i <= 9; i++ )
                threadPool.execute(() -> {
                    System.out.println(Thread.currentThread().getName() + "\t 办理业务" );
                    try {
                        TimeUnit.SECONDS.sleep(1L);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                });
    }

}
```

### 线程池合理配置参数

```shell
# CPU 密集型
	# 意思是该任务需要大量的运算，而没有阻塞，CPU 一直全速运行
	# CPU 密集任务只有在真正的多核 CPU 上才可能得到加速(通过多线程)
	# CPU 密集型任务配置尽可能少的线程数量
	# 一般公式 : CPU 核数 + 1个线程的线程池最大线程数
	
# IO 密集型
	# 由于 IO 密集型任务线程并不是一直在执行任务，则应配置尽可能多的线程
	# 一般公式 : CPU 核数* 2
	
# IO 密集型 2
	# IO 密集型、即该任务需要大量的 IO，即大量的堵塞
	# 在单线程上运行 IO 密集型的任务会导致浪费大量的 CPU 算力浪费在等待上
	# 所以，IO 密集型任务中使用多线程可以大大的加速程序运行，即时在单核 CPU 上
	# 这种加速主要就是利用了被浪费掉的阻塞时间
	# 参考公式 : CPU 核数 / (1 - 阻塞系数)
		# 例: 8 核CPU 8/(1-0.9) = 80 个线程数
```

### 死锁编码以及定位分析

```shell
# 产生死锁的主要原因
	# 死锁是指两个或两个以上的线程在执行过程中，因争夺资源而造成的一种互相等待的现象
	# 例: AB互相拿手枪指着对方，"你先放下枪"，"不，你先放下枪"....
```

![UTOOLS1571117532133.png](https://i.loli.net/2019/10/15/EgHK7N5yCpFAthi.png)

#### 死锁代码验证

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class DeadLockDemo {

    public static void main(String[] args) {
        ShareData data = new ShareData();
        new Thread(() -> {
            data.lock1();
            },"A").start();

        new Thread(() -> {
            data.lock2();
        },"B").start();
    }

}

class ShareData{

    ReentrantLock lock = new ReentrantLock();
    ReentrantLock lock2 = new ReentrantLock();
    public void lock1(){
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 获得锁1");
            TimeUnit.SECONDS.sleep(1);
            lock2();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void lock2(){
        lock2.lock();
        try {
            System.out.println(Thread.currentThread().getName() + "\t 获得锁2");
            lock1();
        } catch (Exception e){
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

}
```

#### 死锁定位分析

```shell
# 找进程
	# jps -l : 列出系统所有正在运行的 Java 进程<ps -ef | grep java>
```

![UTOOLS1571118504233.png](https://i.loli.net/2019/10/15/v7Khp5fyqAL8XO4.png)

 

```shell
# 看问题
	# jstack <进程号>
```

![](http://baijingins.oss-cn-beijing.aliyuncs.com/images/2019/10/15/15711188940981903.png)

