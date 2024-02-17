



# 黑马 - 高性能队列Disruptor

## 1. Disruptor概述

**并发框架Disruptor**

![1699840916419.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699840916419.png)

### 1.1 背景

- 英国外汇交易公司 **LMAX** 开发的一个 **高性能内存队列**
  - 研发初衷是解决 **内存队列的延迟问题（在性能测试中发现竟然与 I/O 操作处于同样的数量级）**
  - 基于 Disruptor 开发的系统单线程能支撑每秒 600w 订单
- 注意：这里说的队列是系统内部的内存队列，而不是 kafka 这样的分布式队列

### 1.2 什么是Disruptor

- Disruptor 是用于一个JVM中多个线程之间的消息队列，作用于ArrayBlokingQueue有相似之处
  - 但从功能、性能都远好于 ArrayBlokingQueue，当多个线程之间传递大量数据或对性能要求较高时，
  - 可以考虑使用 Disruptor 作为 ArrayBlokingQueue 的替代者。
- 官方对 Disruptor 和 ArrayBlockingQueue 的性能在不同的应用场景下做了对比，目测性能有 5-10 倍左右的提升

### 1.3 为什么使用Disruptor

- 传统阻塞的队列使用锁保护线程安全，而锁通过操作系统内核上下文切换实现，会暂停线程去等待锁，直到锁的释放
- 执行这样的上下文切换，会丢失之前保存的数据和指令。由于消费者和生产者之间的速度差异，队列总是接近满或者空的状态，这种状态会导致高水平的写入争用。

#### 1.3.1 传统队列问题

注意：这里说的队列仅限于 java 内部的消息队列

| 队列                  | 有界性 | 锁   | 结构 | 队列类型 |
| --------------------- | ------ | ---- | ---- | -------- |
| ArrayBlockingQueue    | 有界   | 加锁 | 数组 | 阻塞     |
| LinkedBlockingQueue   | 可选   | 加锁 | 链表 | 阻塞     |
| ConcurrentLinkedQueue | 无界   | 加锁 | 链表 | 非阻塞   |
| LinkedTransferQueue   | 无界   | 无锁 | 链表 | 阻塞     |
| PriorityBlockingQueue | 无界   | 加锁 | 堆   | 阻塞     |
| DelayQueue            | 无界   | 加锁 | 堆   | 阻塞     |

#### 1.3.2 Disruptor应用场景

- log4j2
  - 异步日志使用到了disruptor，日志一般是有缓冲区，满了才写到文件，增量追加文件结合NIO等应该也比较快。
  - 所以无论是EventHandler还是WorkHandler处理应该延迟比较小的，写的文件也不多，所以场景是比较适合的。
- Jstorm
  - 在流处理中不同线程中数据交换，数据计算可能蛮多内存中计算，流计算快进快出，disruptor应该是不错的选择
- 百度uid-generator
  - 部分使用 Ring buffer 和去伪共享等思路缓存已生成的 uid，应该部分也参考了 disruptor

### 1.4 Disruptor核心概念

> 先了解Disruptor的核心概念，来了解它是如何运作的。下面介绍的概念模型，既是领域对象，也是映射到代码实现上的核心对象。

#### 1.4.1 Ring Buffer

> Disruptor中的数据结构，用于存储生产者生产的数据

- 如其名，环形的缓冲区，曾经 RingBuffer 是 Disruptor 中的最主要对象，但从3.0x后，其职责被简化为仅仅负责对通过 Disruptor 进行交换的数据（事件）进行存储和更新。
- 在一些更高级的应用场景中，Ring Buffer 可以由用户的自定义实现来完全替代。

#### 1.4.2 Sequence

> 序号，在Disruptor框架中，任何地方都有序号

- 生产者生产的数据放在 RingBuff 中哪个位置、消费者应该消费哪个位置的数据、RingBuffer中某个位置的数据是什么？这些都是由序号来决定的。
- 序号可以简单理解为一个 AtomicLong 类型变量，**它使用了padding的方法去消除缓存的伪共享问题**

#### 1.4.3 Sequencer

> 序号生成器，这个类主要是用来协调生产者的

- 在生产者生产数据的时候，Sequencer会产生一个可用的序号（Sequence），然后生产者就知道数据放在环形队列的哪个位置了。
- **Sequencer是Disruptor的真正核心，此接口有两个实现类：**
  - **SingleProducerSequencer**
  - **MultiProducerSequencer**
  - **它们定义在生产者和消费者之间快速、正确地传递数据的并发算法**

#### 1.4.4 Sequence Barrier

> 序号屏障

- 由它告诉消费者一个序号，该消费者只能去消费该序号位置上的数据，要是没有数据，就在此等待

#### 1.4.5 Wait Strategy

> Wait Strategy 决定了一个消费者怎么等待生产者将事件（Event）放入Disruptor中

- 设想场景：消费 > 生产，此时消费者该怎样进行等待？WaitStrategy 就是为了解决问题而诞生的

#### 1.4.6 Event

- 从生产者到消费者传递的数据叫做Event。它不是一个被 Disruptor 定义的特定类型，而是由 Disruptor 的使用者定义并指定

#### 1.4.7 EventHandler

- Disruptor 定义的事件处理接口，由用户实现，用于处理事件，是 Consumer 的真正实现

#### 1.4.8 Producer

- 即生产者，只是泛指调用 Disruptor 发布事件的用户代码，Disruptor 没有定义特定接口或类型

### 1.5 Disruptor特性

> Disruptor就像一个队列一样，用于在不同的线程之间迁移数据，但是它也实现了一些其它队列没有的特性，如：

- 同一个"事件"可以有多个消费者，消费者之间既可以并行处理，也可以相互依赖形成处理的先后次序（形成一个依赖图）
- 预分配用于存储事件内容的内存空间
- 针对极高的性能目标而实现的极度优化和无锁的设计

## 2. Disruptor入门

> 使用一个简单例子来体验Disruptor，生产者会传递一个long类型的值到消费者，消费者接受这个值后会打印出这个值。

### 2.1 添加依赖

```xml
<!-- https://mvnrepository.com/artifact/com.lmax/disruptor -->
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>4.0.0</version>
</dependency>
```

### 2.2 Disruptor API

> Disruptor的API十分简单，主要有以下几个步骤

#### 2.2.1 定义事件

> 首先创建一个 **LongEvent** 类，这个类将会被放入环形队列中作为消息内容。
>
> 事件（Event）就是通过 Disruptor 进行交换的数据类型

```java
@Data
public class LongEvent {
    
    private long value;
    
}
```

#### 2.2.2 定义事件工厂

> 为了使用Disruptor的内存预分配event，我们需要定义一个EventFactory

- EventFactory 定义了如何实例化 Event 至 RingBuffer，需要实现接口 EventFactory<T>
- Event = "数据槽"
  1. 发布者 get event by RingBuffer
  2. 发布者 fill event
  3. 发布者 push event to RingBuffer
  4. 消费者 get event by RingBuffer

```java
import com.lmax.disruptor.EventFactory;

public class LongEventFactory implements EventFactory<LongEvent> {
    @Override
    public LongEvent newInstance() {
        return new LongEvent();
    }
}
```

#### 2.2.3 定义事件处理的具体实现

> 为了让消费者处理这些事件，所以我们在这里定义一个事件处理器，负责打印event
>
> 通过实现接口 EventHandler<T> 定义事件处理的具体实现
>
> 后面做性能测试的时候，可以把这个打印注释掉

```java
import cn.hutool.core.util.StrUtil;
import com.lmax.disruptor.EventHandler;

public class LongEventHandler implements EventHandler<LongEvent> {
    @Override
    public void onEvent(LongEvent event, long sequence, boolean endOfBatch) {
        System.out.println(StrUtil.format("Consumer: {}, Event: value={}, sequence={}", Thread.currentThread().getName(), event.getValue(), sequence));
    }
}
```

#### 2.2.4 指定等待策略

> Disruptor定义了 WaitStrategy 接口用于抽象 Consumer 如何等待新事件，这是策略模式的应用

```java
WaitStrategy YIELDING_WAIT = new YieldingWaitStrategy();
```

#### 2.2.5 启动Disruptor

> 注意：RingBufferSize的大小必须是2^n

```java
import com.lmax.disruptor.YieldingWaitStrategy;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import java.util.concurrent.Executors;

public class LongEventMain {

    public static void main(String[] args) {
        Disruptor<LongEvent> disruptor = new Disruptor<>(new LongEventFactory(), // 指定事件工厂
                1024, // 指定RingBufferSize
                Executors.defaultThreadFactory(), // 默认线程池
                ProducerType.SINGLE, // 单线程模式,获得额外性能
                new YieldingWaitStrategy() // 指定等待策略
        );
        // 设置消费者
        disruptor.handleEventsWith(new LongEventHandler());

        // 启动disruptor线程
        disruptor.start();
    }

}
```

#### 2.2.6 使用Translators发布事件

> 在Disruptor3.0x后，加入了丰富的lambda风格api，用于简化开发流程。
>
> 所以在3.0x后，首选使用Event Publisher/Event Translator来发布事件。

```java
import com.lmax.disruptor.EventTranslatorOneArg;
import com.lmax.disruptor.RingBuffer;
import lombok.AllArgsConstructor;

@AllArgsConstructor
public class LongEventProducerWithTranslator {

    private final RingBuffer<LongEvent> ringBuffer;

    private static final EventTranslatorOneArg<LongEvent, Long> TRANSLATOR
            = (event, sequence, data) -> event.setValue(data);

    public void onData(Long data) {
        ringBuffer.publishEvent(TRANSLATOR, data);
    }

}
```

#### 2.2.7 关闭Disruptor

```java
// 关闭disruptor,方法会阻塞,直至所有事件都得到处理
disruptor.shutdown();
```

### 2.3 代码整合

#### 2.3.1 LongEventMain

> 消费者-生产者 启动类，其依靠构造Disruptor对象，调用start()方法完成启动线程。
>
> Disruptor需要RingBuffer环、消费者数据处理工厂、WaitStrategy等

```java
import cn.hutool.core.util.RandomUtil;
import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.YieldingWaitStrategy;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import java.util.concurrent.Executors;

public class LongEventMain {

    public static void main(String[] args) {
        Disruptor<LongEvent> disruptor = new Disruptor<>(new LongEventFactory(), // 指定事件工厂
                1024, // 指定RingBufferSize
                Executors.defaultThreadFactory(), // 默认线程池
                ProducerType.SINGLE, // 单线程模式,获得额外性能
                new YieldingWaitStrategy() // 指定等待策略
        );
        // 设置消费者
        disruptor.handleEventsWith(new LongEventHandler());

        // 启动disruptor线程
        disruptor.start();

        // 获取RingBuffer环,用于接收生产者生产的事件
        RingBuffer<LongEvent> ringBuffer = disruptor.getRingBuffer();
        // 为RingBuffer指定事件生产者
        LongEventProducerWithTranslator producer = new LongEventProducerWithTranslator(ringBuffer);
        // 循环遍历
        for (int i = 0; i < 100; i++) {
            // 发布随机数
            producer.onData(RandomUtil.randomLong(1, 999999));
        }
        // 关闭disruptor,方法会阻塞,直至所有事件都得到处理
        disruptor.shutdown();
    }

}
```

### 2.3.2 运行测试

> 测试结果部分截图

![1699865446457.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699865446457.png)

## 3. 性能对比测试

> 为了直观感受Disruptor有多快，设计一个性能对比测试：
>
> Producer发布1亿次事件，从发布第一个事件开始计时，捕捉Consumer处理完所有事件的耗时
>
> 测试用例在Producer如何将事件通知到Consumer的实现方式上，设计了两种不同的实现:

1. Producer的事件发布和Consumer的事件处理在不同线程，通过ArrayBlockingQueue传递给Consumer进行处理
2. Producer的事件发布和Consumer的事件处理在不同线程，通过Disruptor传递给Consumer进行处理

### 3.1 代码实现

#### 3.1.1 计算代码

> 进行CAS累加运算

```java
public class CommonUtil {

    private static final AtomicLong count = new AtomicLong(0);

    public static void calculation() {
        count.incrementAndGet();
    }

    public static long get() {
        return count.get();
    }

}
```

#### 3.1.2 抽象类

> 进行1亿次CAS运算计算耗时

```java
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.date.TimeInterval;
import cn.hutool.core.util.StrUtil;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public abstract class AbstractTask <T> {

    /**
     * 线程池
     */
    private static final ExecutorService executor = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() + 1);

    private static final CountDownLatch countDownLatch = new CountDownLatch(1);

    /**
     * 1亿次测试
     */
    public static long taskSize = 100000000;

    /**
     * 测试方法
     */
    public void invoke() {
        // 开启计时器
        TimeInterval timer = DateUtil.timer();
        Runnable monitor = monitor();
        if (null != monitor) {
            executor.execute(monitor);
        }
        // 启动
        start();
        // 执行任务发布
        Runnable runnable = getTask();
        for (int i = 0; i < taskSize; i++) {
            runnable.run();
        }

        // 停止
        stop();
        // 等待任务发布完成
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        executor.shutdown();

        // 获取处理结果
        T result = getResult();
        // 计算耗时
        long interval = timer.interval();
        // 计算吞吐量
        long throughput = (taskSize / interval) * 1000;
        System.out.println(StrUtil.format("每秒吞吐量:[{}], ({}/{})", throughput, result, interval));
    }

    /**
     * 获取监听器
     */
    public Runnable monitor() {
        return null;
    }

    /**
     * 启动任务
     */
    public void start() {

    }

    /**
     * 完成任务
     */
    public void complete() {
        countDownLatch.countDown();
    }

    /**
     * 停止任务
     */
    public void stop() {

    }

    /**
     * 获取需要执行的任务
     */
    public abstract Runnable getTask();

    /**
     * 获取运行结果
     */
    public abstract T getResult();

}
```

#### 3.1.3 Disruptor性能测试代码

```java
import cn.hutool.core.util.RandomUtil;
import com.lmax.disruptor.YieldingWaitStrategy;
import com.lmax.disruptor.dsl.Disruptor;
import com.lmax.disruptor.dsl.ProducerType;

import java.util.concurrent.Executors;

public class DisruptorTest extends AbstractTask<Long> {

    private Disruptor<LongEvent> disruptor = null;
    private LongEventProducerWithTranslator translator = null;

    @Override
    public void start() {
        // 构建Disruptor
        disruptor = new Disruptor<>(new LongEventFactory(),
                1024 * 1024,
                Executors.defaultThreadFactory(),
                ProducerType.SINGLE,
                new YieldingWaitStrategy());

        // 配置事件处理类
        disruptor.handleEventsWith(new LongEventHandler());
        // 启动
        disruptor.start();
        // 创建事件发布对象
        translator = new LongEventProducerWithTranslator(disruptor.getRingBuffer());
    }

    @Override
    public void stop() {
        disruptor.shutdown();
        complete();
    }

    @Override
    public Runnable getTask() {
        return this::publishEvent;
    }

    @Override
    public Long getResult() {
        return CommonUtil.get();
    }

    private void publishEvent() {
        CommonUtil.calculation();
        // 发布随机数
        translator.onData(RandomUtil.randomLong(0, 999999));
    }

    public static void main(String[] args) {
        DisruptorTest disruptorTest = new DisruptorTest();
        disruptorTest.invoke();
    }

}
```

> 输出结果

![1699867704369.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699867704369.png)

#### 3.1.4 ArrayBlockingQueue性能测试代码

```java
import cn.hutool.core.util.RandomUtil;
import java.util.concurrent.ArrayBlockingQueue;

public class ArrayBlockingQueueTest extends AbstractTask<Long> {

    private static final ArrayBlockingQueue<Long> queue = new ArrayBlockingQueue<>(10000000);

    @Override
    public Runnable monitor() {
        return () -> {
            for (int i = 0; i < taskSize; i++) {
                try {
                    // 获取一个元素
                    queue.take();
                    // 执行计算
                    CommonUtil.calculation();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            complete();
        };
    }

    @Override
    public Runnable getTask() {
        return this::publish;
    }

    @Override
    public Long getResult() {
        return CommonUtil.get();
    }

    public void publish() {
        try {
            queue.put(RandomUtil.randomLong(0, 999999));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ArrayBlockingQueueTest test = new ArrayBlockingQueueTest();
        test.invoke();
    }

}
```

> 输出结果

![1699867668287.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699867668287.png)

### 3.2 测试对比

| 测试类             | 运算次数 | 耗时(ms) | 吞吐量/s |
| ------------------ | -------- | -------- | -------- |
| ArrayBlockingQueue | 1亿次    | 17992    | 5558000  |
| Disruptor          | 1亿次    | 3499     | 28579000 |

### 3.3 Disruptor官方性能测试

> Disruptor论文中讲述了一个实验：

- 测试函数：对一个64位的计数器自增5亿次
- 机器环境：2.4g 6c

| Method             | Time(ms) |
| ------------------ | -------- |
| 单线程             | 300      |
| 单线程使用CAS      | 5700     |
| 单线程使用锁       | 10000    |
| 单线程使用volatile | 4700     |
| 多线程使用CAS      | 30000    |
| 多线程使用锁       | 224000   |

## 4. 高性能原理

- 引入环形的数据结构：数组元素不会被回收，避免频繁GC
- 无锁的设计：采用CAS无锁方式，保证线程的安全性
- 属性填充：通过添加额外的无用信息，避免伪共享问题
- 元素位置的定位：采用跟一致性哈希一样的方式，一个索引，进行自增

### 4.1 伪共享概念

#### 4.1.1 计算机缓存构成

> 下图是计算的基本结构。L1、L2、L3（一级/二级/三级缓存）越靠近CPU的缓存，速度越快，容量也越小。
>
> L1紧靠使用它的CPU内核、L2被单独的CPU核使用、L3被单个插槽上的所有CPU核共享。
>
> 最后是主存，由全部插槽上的所有CPU核共享。

![1699868895271.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/1699868895271.png)

> 当CPU读取数据时，L1 -> L2 -> L3 -> 主存，依次查找。
>
> 一般来说，每级缓存的命中率大概都在80%左右，也就是说，L1中能找到到80%，剩下的20%总数才需要从L2、L3、主存中找。
>
> 由此可见，L1是CPU缓存架构中最为重要的部分。

![disruptor03.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor03.png)

> 下表是一些缓存未命中的消耗数据：

| 从CPU到 | 大约需要的CPU周期 | 大约需要的时间 |
| ------- | ----------------- | -------------- |
| 主存    |                   | 约60-80ms      |
| QPI总线 |                   | 约20ms         |
| L3      | 约40-45cycles     | 约15ns         |
| L2      | 约10cycles        | 约3ns          |
| L1      | 约3-4cycles       | 约1ns          |
| 寄存器  | 1cycle            |                |

> 可见CPU读取主存中的数据会比L1中读取慢了近2个数量级。

#### 4.1.2 什么是缓存行

> 为了解决计算机系统中主存与CPU之间运行速度差的问题，会在CPU与主存之间添加多级高速缓存
>
> 下图所示是两级Cache结构

![disruptor05.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor05.png)

> Cache内部是按行存储的，其中每一行称为一个 `cache line`，cache line是Cache和RAM交换数据的最小单位
>
> cache line的大小一般为2^n字节，通常为64byte

![disruptor06.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor06.png)

> 当CPU把内存数据载入Cache时，会把临近的共64byte的数据一同放入同一个cache line
>
> 因为空间局部性：临近的数据在将来被访问的可能性大

```shell
# Linux查看缓存行大小
more /sys/devices/system/cpu/cpu1/cache/index0/coherency_line_size

# 通常结果为
64
```

#### 4.1.3 什么是共享

> Cache line有效地引用内存中的一块地址，一个Java的long类型是8个byte，一个Cache line可以缓存8个long类型的变量
>
> 当访问一个long数组时，当数组中的一个值被加载到缓存中，它会额外加载另外7个，实现非常快速的遍历这个数组
>
> 事实上，都可以非常快速的遍历在连续的内存块中分配的任意数据结构，反之速度就会慢（如链表在内存中不是彼此相邻的）

![disruptor04.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor04.png)

如上，表面上X和Y是被独立线程操作的，而且两操作之间也没有任何关系，只不过它们共享了一个Cache line，但所有竞争冲突都来源于共享

#### 4.1.4 什么是伪共享

> 当CPU访问变量时，按 高速缓存 -> 主存 的顺序读取，从主存中读时会将该变量所在的Cache line拷贝到高速缓存中
>
> 由于存放到Cache line的是内存块而不是单个变量，所以可能会把多个变量存放到一个Cache line中
>
> 当多个线程同时修改一个Cache line里的多个变量时，由于同时只能有一个线程操作Cache line，相比每个变量放到一个Cache line性能就有所下降，这就是伪共享。

![disruptor07.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor07.png)

如上，x,y同时被放到了CPU的一级和二级缓存，当线程1使用CPU1对x进行更新时，首先会修改CPU1的L1中的x所在Cache line

此时，缓存一致性协议会导致CPU2中变量x对应的Cache line失效，那么线程2更新变量y时，就只能从L2查找，更坏的情况就是L3、主存

**归根结底：缓存是以Cache line作为一个单位，失效x时，同Cache line中的y也会失效，反之亦然**

#### 4.1.5 为何会出现伪共享

> 伪共享的产生是因为多个变量被放入了一个缓存行，并且多个线程同时去写入缓存行中不同变量。
>
> 多个变量之所以会被放到一个Cache line，是因为Cache与内存交换数据的单位就是Cache line，当CPU要访问的变量没有在Cache命中时，根据程序运行的局部性原理，会将该变量在内存中大小为Cache line的内存放入Cache line

```java
long a;
long b;
long c;
long d;
```

如上，声明4个long变量，假设Cache line的大小为32个byte，那么当CPU访问变量a时，发现没有在Cache命中，就会从主存把变量a以及其内存地址附近的b,c,d放入Cache line。

单线程下多个变量放入Cache line对性能有影响吗？

正常情况下，单线程访问时由于数组元素被放入到一个或多个Cache line对代码执行时有利的，数据都在缓存中，又没有其他线程操作该Cache lien导致其失效，程序执行自然更快。

#### 4.1.6 如何解决伪共享

> 解决伪共享最直接的方法就是填充（padding）
>
> 例如：下面的VolatileLong，一个long占8个byte，java的对象头占8个byte(32位系统) 或者 12byte(64位系统，默认开启对象头压缩，不开启占16个byte)。一个Cache line 64 byte，那我们可以填充6个long (6 * 8 = 48 byte)

##### 不使用字段填充

```java
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;

@NoArgsConstructor
@AllArgsConstructor
public class VolatileData {

    /**
     * 占用8byte，+ 48 + 对象头 = 64byte
     */
    volatile long value;

    public long accumulationAdd() {
        // 因为单线程操作不需要加锁
        return value + 1;
    }

    public long getValue() {
        return value;
    }

}
```

> 内存布局

![disruptor22.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor22.png)

##### 填充字段

> JDK1.7以后，就会自动优化代码删除无用代码，在JDK1.7以后的版本，这些不生效了

```java
/**
 * 缓存行填充父类
 */
public class DataPadding {

    /**
     * 填充5个long类型变量，8*5=40byte
     * jvm优化：删除无用代码
     */
    private long p1, p2, p3, p4, p5;

    /**
     * 需要操作的数据
     */
    volatile long value;

}
```

![disruptor23.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor23.png)

##### 继承的方式

```java
/**
 * 缓存行填充父类
 */
public class DataPadding {
  // 填充5个long类型变量，8*5=40个字节
  private long p1, p2, p3, p4, p5;
}
```

###### 继承缓存填充类

```java
/**
 * 继承DataPadding
 */
@Data
public class VolatileData extends DataPadding {
  // 占用8byte + 48 + 对象头 = 64byte
  volatile long value;
  public long accumulationAdd() {
    // 因为单线程操作不需要加锁
    return value + 1;
  }

  public long getValue() {
    return value;
  }
}
```

##### Disruptor填充方式

```java
class LhsPadding {
  protected long p1, p2, p3, p4, p5, p6, p7;
}

class Value extends LhsPadding {
  protected volatile long value;
}

class RhsPadding extends Value {
  protected long p9, p10, p11, p12, p13, p14, p15;
}
```

###### 继承填充类

```java
/**
 * 继承DataPadding
 */
@Data
public class VolatileData extends RhsPadding {
  // 占用8byte + 48 + 对象头 = 64byte
  volatile long value;
  public long accumulationAdd() {
    // 因为单线程操作不需要加锁
    return value + 1;
  }

  public long getValue() {
    return value;
  }
}
```

> 内存布局

![disruptor27.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor27.png)

##### @Contended注解

```java
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.TYPE})
public @interface Contended {
  String value() default "";
}
```

###### 注解填充类

```java
@Contended
@Data
public class VolatileData {
  // 占用8byte + 48 + 对象头 = 64byte
  volatile long value;
  public long accumulationAdd() {
    // 因为单线程操作不需要加锁
    return value + 1;
  }

  public long getValue() {
    return value;
  }
}
```

> 内存布局

![disruptor26.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor26.png)

##### 注意事项

> JDK8中提供了 @sum.misc.Contended 来避免伪共享时，在运行时需要设置JVM启动参数 -XX:RestrictContended 否则可能不生效

#### 4.1.7 性能对比

##### 4.1.7.1 测试数据

> 使用和不使用缓存行填充的对比

| 对象          | NoPadding(MS) | DataPadding(MS) | RhsPadding(MS) | Contended(MS) |
| ------------- | ------------- | --------------- | -------------- | ------------- |
| volatileData1 | 3751          | 1323            | 1307           | 1291          |
| volatileData2 | 3790          | 1383            | 1311           | 1314          |
| volatileData3 | 7551          | 1400            | 1311           | 1333          |
| volatileData4 | 7669          | 1407            | 1317           | 1356          |
| volatileData5 | 8577          | 1447            | 1327           | 1361          |
| volatileData6 | 8705          | 1479            | 1339           | 1375          |
| volatileData6 | 8741          | 1512            | 1368           | 1389          |

#### 4.1.8 Disruptor解决伪共享

> 在Disruptor中有一个重要的类Sequence，该类包装了一个volatile修饰的long类型数据value，无论是Disruptor中的基于数组实现的缓冲区RingBuffer，还是生产者、消费者，都有各自独立的Sequence，RingBuffer缓冲区中，Sequence标示着写入进度，例如每次生产者要写入数据进缓冲区时，都要调用RingBuffer.next()来获得下一个可使用的相对位置。
>
> 对于生产者和消费者来说，Sequence标示着它们的事件序号，来看看Sequence类的源码：

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

class LhsPadding
{
    protected byte
        p10, p11, p12, p13, p14, p15, p16, p17,
        p20, p21, p22, p23, p24, p25, p26, p27,
        p30, p31, p32, p33, p34, p35, p36, p37,
        p40, p41, p42, p43, p44, p45, p46, p47,
        p50, p51, p52, p53, p54, p55, p56, p57,
        p60, p61, p62, p63, p64, p65, p66, p67,
        p70, p71, p72, p73, p74, p75, p76, p77;
}

class Value extends LhsPadding
{
    protected long value;
}

class RhsPadding extends Value
{
    protected byte
        p90, p91, p92, p93, p94, p95, p96, p97,
        p100, p101, p102, p103, p104, p105, p106, p107,
        p110, p111, p112, p113, p114, p115, p116, p117,
        p120, p121, p122, p123, p124, p125, p126, p127,
        p130, p131, p132, p133, p134, p135, p136, p137,
        p140, p141, p142, p143, p144, p145, p146, p147,
        p150, p151, p152, p153, p154, p155, p156, p157;
}

/**
 * Concurrent sequence class used for tracking the progress of
 * the ring buffer and event processors.  Support a number
 * of concurrent operations including CAS and order writes.
 *
 * <p>Also attempts to be more efficient with regards to false
 * sharing by adding padding around the volatile field.
 */
public class Sequence extends RhsPadding
{
    static final long INITIAL_VALUE = -1L;
    private static final VarHandle VALUE_FIELD;

    static
    {
        try
        {
            VALUE_FIELD = MethodHandles.lookup().in(Sequence.class)
                    .findVarHandle(Sequence.class, "value", long.class);
        }
        catch (final Exception e)
        {
            throw new RuntimeException(e);
        }
    }

    /**
     * Create a sequence initialised to -1.
     */
    public Sequence()
    {
        this(INITIAL_VALUE);
    }

    /**
     * Create a sequence with a specified initial value.
     *
     * @param initialValue The initial value for this sequence.
     */
    public Sequence(final long initialValue)
    {
        VarHandle.releaseFence();
        this.value = initialValue;
    }

    /**
     * Perform a volatile read of this sequence's value.
     *
     * @return The current value of the sequence.
     */
    public long get()
    {
        long value = this.value;
        VarHandle.acquireFence();
        return value;
    }

    /**
     * Perform an ordered write of this sequence.  The intent is
     * a Store/Store barrier between this write and any previous
     * store.
     *
     * @param value The new value for the sequence.
     */
    public void set(final long value)
    {
        VarHandle.releaseFence();
        this.value = value;
    }

    /**
     * Performs a volatile write of this sequence.  The intent is
     * a Store/Store barrier between this write and any previous
     * write and a Store/Load barrier between this write and any
     * subsequent volatile read.
     *
     * @param value The new value for the sequence.
     */
    public void setVolatile(final long value)
    {
        VarHandle.releaseFence();
        this.value = value;
        VarHandle.fullFence();
    }

    /**
     * Perform a compare and set operation on the sequence.
     *
     * @param expectedValue The expected current value.
     * @param newValue      The value to update to.
     * @return true if the operation succeeds, false otherwise.
     */
    public boolean compareAndSet(final long expectedValue, final long newValue)
    {
        return VALUE_FIELD.compareAndSet(this, expectedValue, newValue);
    }

    /**
     * Atomically increment the sequence by one.
     *
     * @return The value after the increment
     */
    public long incrementAndGet()
    {
        return addAndGet(1);
    }

    /**
     * Atomically add the supplied value.
     *
     * @param increment The value to add to the sequence.
     * @return The value after the increment.
     */
    public long addAndGet(final long increment)
    {
        return (long) VALUE_FIELD.getAndAdd(this, increment) + increment;
    }

    /**
     * Perform an atomic getAndAdd operation on the sequence.
     *
     * @param increment The value to add to the sequence.
     * @return the value before increment
     */
    public long getAndAdd(final long increment)
    {
        return (long) VALUE_FIELD.getAndAdd(this, increment);
    }

    @Override
    public String toString()
    {
        return Long.toString(get());
    }
}
```

> 从源码看出，真正使用到的变量value，前后空间都被byte填满，达到刚好满足一个Cache line 64byte的大小。
>
> 保证每次处理数据时都不会与其他变量发生冲突。

### 4.2 无锁的设计

#### 4.2.1 锁机制存在的问题

- 在多线程竞争下，加锁、释放锁会导致比较多的上下文切换和调度延时，引起性能问题，而且在上下文切换的时候
  - CPU之前缓存的指令和数据都将失效，对性能有很大的损失，用户态的锁虽然避免了这些问题，
  - 但其实只是在它们没有真实的竞争时才有效。
- 一个线程持有锁会导致其他所有需要此锁的线程挂起直至该锁释放。
- 如果一个优先级高的线程等待一个优先级低的线程释放锁，会导致优先级反转（Priority Inversion）引起性能风险

#### 4.2.2 CAS无锁算法

> 实现无锁(lock-free)的非阻塞算法有多种实现方式，其中CAS是一种乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，其他线程都失败。
>
> 失败的线程并不会被挂起，而是被告知此次竞争失败，可以再次尝试。CAS是一个CPU级别的指令。

![disruptor08.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor08.png)

**注意：**

这可以是CPU的两个不同核心，但不会是两个独立的CPU。

CAS操作比锁消耗资源少的多，因为它们不涉及操作系统，直接在CPU上操作。

但并非没有代价，在上面的试验中，单线程无锁耗时300ms，单线程有锁耗时10000ms，单线程使用CAS耗时5700ms。

比使用锁耗时少，比不需要考虑竞争的单线程耗时多。

#### 4.2.3 传统队列问题

> 队列的底层数据结构一般分为三种：`数组、链表、堆`。其中堆是为了实现带有优先级特性的队列，暂且不考虑。
>
> 在稳定性和性能要求特别高的系统中，为了防止生产者速度过快，导致内存溢出，只能选择有界队列。
>
> 同时，为了减少Java的垃圾回收对系统性能的影响，会尽量选择将 array/heap 格式的数据结构。这样筛选下来，符合条件的队列就只有ArrayBlockingQueue，但是它是通过加锁的方式保证线程安全、还存在伪共享问题。这两者严重影响性能。

##### Disruptor的无锁设计

> 多线程环境下，多个生产者通过 do/while 循环的条件CAS，来判断每次申请的空间是否已经被其他生产者占据。
>
> 假如已经被占据，该函数会返回失败，While循环重新执行，申请写入空间。

```java
do {
  current = cursor.get();
  next = current + n;
  if (!hasAvailableCapacity(gatingSequences, n, current)) {
    throw InsufficientCapacityException.INSTANCE;
  }
}
while (!cursor.compareAndSet(current, next));
// next类比于ArrayBlockingQuene的数组索引index
return next;
```

### 4.3 环形数组结构

> 环形数组结构是整个Disruptor的核心所在。

#### 4.3.1 什么是环形数组

> RingBuffer是一个环，用作在不同线程上下文间传递数据的buffer，RingBuffer拥有一个序号，这个序号指向数组中下一个可用元素。

![disruptor09.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor09.png)

#### 4.3.2 为什么使用环形数组

> 为了避免垃圾回收，采用数组而非链表，同时，数组对处理器的缓存机制更加友好。
>
> 首先数组比链表快，而且相邻的数组元素也是会被预加载的，可以减少CPU读L1后的缓存次数。
>
> 还可以为数组预分配内存，使得数组对象一直存在（除非程序终止）这样可以减少GC。
>
> 数组大小为2^n，元素定位可以通过位运算效率更高。
>
> 其实本质就是一个普通数组，只是在数据放满之后，再填充数据又从0开始，覆盖之前数据，就形成一个环。

### 4.4 元素位置定位

数组长度2^n，通过位运算，加快定位的速度，下标采取递增的形式，不用担心index溢出的问题，index是long类型，即使100wQPS的处理速度，也需要30w年才能用完。

### 4.5 等待策略

> 定义Consumer如何进行等待下一个事件的策略。
>
> 根据实际运行环境的CPU硬件特点选择恰当的策略，并配合特定的JVM配置参数，能够实现不同的性能提升。

#### BlockingWaitStrategy

**默认策略**，内部是使用锁和condition来控制线程的唤醒，是最低效的策略。

但其对CPU消耗最小，且在各种不同部署环境中能提供更加一致的性能表现。

#### SleepingWaitStrategy

性能跟BlockingWaitStrategy差不多，对CPU的消耗也类似，但其对生产者线程的影响最小

通过使用 `LockSupport.parkNanos(1)` 来实现循环等待，适合用于异步日志类似的场景。

#### YieldingWaitStrategy

是可以是使用在低延迟系统的策略之一，将自旋以等待序列增加到合适的值。

在循环体内，将调用 `Thread.yield()` 以允许其他排队的线程运行。

在要求极高性能且处理线程数小于CPU逻辑核心数的场景中，推荐使用此策略。

#### BusySpinWaitStrategy

**性能最好**，适用于低延迟的系统，在要求极高性能且事件处理线程数小于CPU逻辑核心数的场景中，推荐使用此策略;

#### PhasedBackoffWaitStrategy

自旋 + yield + 自定义策略，适合CPU资源紧缺，吞吐量和延迟并不高的场景

## 5. 生产和消费模式

> 根据上面环形结构，来分析一下Disruptor的工作原理。
>
> Disruptor不像传统队列，分为一个队头指针和一个队尾指针，而且只有一个角标（sequence），那是如何保证生产的消息不会覆盖没有消费掉的消息呢？
>
> Disruptor中生产者分为单生产者和多生产者，而消费者没有区分。
>
> 单生产者情况下，就是普通的生产者向RingBuffer中放置数据，消费者获得最大可消费的位置，并进行消费。
>
> 而多生产者时，又多出一个跟RingBuffer同样大小的Buffer，称为AvailableBuffer。
>
> 在多生产者中，每个生产者首先通过CAS竞争获取可以写的空间，然后再往里放数据，如果此时消费者要消费数据，那每个消费者都需要获取最大可消费的下标，这个下标就是在 AvailableBuffer 进行获取得到的最长连续的序列下标。

### 5.1 单生产者生产数据

> 生产者单线程写数据的流程比较简单：

1. 申请写入m个元素
2. 若是有m个元素可以写入，则返回最大的序列号。这主要是判断是否会覆盖未读的元素。
3. 若是返回正确，则生产者开始写入元素。

![disruptor11.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor11.png)

### 5.2 多生产者生产数据

> 多个生产者情况下，会遇到 `如何防止多个线程重复写同一个元素` 的问题。
>
> Disruptor的解决方法是：每个线程获取不同的一段数组空间进行操作，这个通过CAS很容易做到。（分配元素时，用CAS判断下这段空间是否被分配出去即可）
>
> 但是会遇到一个新问题：`如何防止读取的时候，读到还未写的元素`。
>
> Disruptor在多个生产者情况下，引入一个与RingBuffer同样大小的AvailbaleBuffer。当某个位置写入成功时，便把AvailableBuffer相应的位置置为写入成功。读取的时候，会遍历AvaiableBuffer，来判断元素是否已经就绪。

#### 5.2.1 生产流程

1. 申请写入m个元素
2. 若是有m个元素可以写入，则返回最大的序列号。每个生产者会被分配一段独享的空间
3. 生产者写入元素，写入元素的同时设置AvaiableBuffer里相应的位置，以标记哪些位置是写入成功的。

![disruptor12.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor12.png)

> 如上图所示，Write1和Write2两个线程写入数组，都申请可写的数组空间。
>
> Write1被分配3-5，Write2被分配6-9
>
> Write1写入3，同时将AvaiableBuffer相应位置置位，标记写入成功，往后移一位，开始写4。Write2同理，最终都写入完成。

#### 5.2.2 CAS检测空间占用

> 防止不同生产者对同一段空间内写入的代码，如下所示：

```java
public long tryNext(int n) throws InsufficientCapacityException {
  if (n < 1) {
    throw new IllegalArgumentException("n num be > 0");
  }
  long current;
  long next;
  do {
    current = cursor.get();
    next = current + n;
    if (!hasAvaibaleCapacity(gatingSequences, n, current)) {
      throw InsufficientCapacityException.INSTANCE;
    }
  } while (!cursor.compareAndSet(current, next));
  return next;
}
```

### 5.3 多生产者消费数据

> 绿色代表写入完成的数据。
>
> 假设三个生产者在写中，还没有置位AvailableBuffer，那消费者可获取的消费下标只能获取到6，然后等生产者都写入完成后，通知到消费者，消费者继续重复上面的步骤。

#### 5.3.1 消费流程

1. 申请读取到序号n
2. 若 writer cursor >= n，此时仍无法确定连续可读的最大下标。从 reader cursor 开始获取AvaiableBuffer，一直到第一个不可用的元素，然后返回最大连续可读元素的位置
3. 消费者读取元素

![disruptor13.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor13.png)

> 如上图所示，读线程读到下标为2的元素，3个写线程正在向RingBuffer相应位置写数据，写线程被分配到的最大元素下标是11
>
> 读线程申请读取到下标从3到11的元素，判断 writer cursor >= 11。然后开始读取AvaiableBuffer，从3开始，往后读取，发现下标7元素没有写入完成，于是WaitFor(11)发挥6
>
> 然后消费者读取下标从3到6共4个元素。

## 6. 高级使用

### 6.1 并行模式

#### 6.1.1 单一写者模式

> 在并发系统中提高性能最好的方式之一就是单一写者原则。
>
> 如果业务系统中，只有一个事件生产者，那么可以设置为单一生产者模式来提高系统的性能。

![disruptor14.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor14.png)

```java
public class singleProductorLongEventMain {
  public static void main(String[] args) throws Exception {
    Disruptor<LongEvent> disruptor = new Disruptor<>(new LongEventFactory(), // 指定事件工厂
                1024, // 指定RingBufferSize
                Executors.defaultThreadFactory(), // 默认线程池
                ProducerType.SINGLE, // 单线程模式,获得额外性能
                new YieldingWaitStrategy() // 指定等待策略
        );
  }
}
```

#### 6.1.2 串行消费

> 比如：现在触发一个注册Event，需要有一个Handler来存储信息，一个Handler来发邮件等.

![disruptor15.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor15.png)

```java
/**
 * 串行依次执行
 */
public static void serial(Disruptor<LongEvent> disruptor) {
  disruptor.handleEventsWith(new C11EventHandler()).then(new C21EventHandler());
  disruptor.strat();
}
```

#### 6.1.3 菱形方块执行

![disruptor16.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor16.png)

```java
public static void diamond(Disruptor<LongEvent> disruptor) {
  disruptor.handleEventsWith(new C11EventHandler(), new C12EventHandler())
    .then(new C21EventHandler());
  disruptor.strat();
}
```

#### 6.1.4 链式并行计算

![disruptor17.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor17.png)

```java
public static void chain(Disruptor<LongEvent> disruptor) {
  disruptor.handleEventsWith(new C11EventHandler()).then(new C21EventHandler());
  disruptor.handleEventsWith(new C12EventHandler()).then(new C22EventHandler());
  disruptor.strat();
}
```

#### 6.1.5 相互隔离模式

![disruptor18.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor18.png)

```java
public static void parallelWithPool(Disruptor<LongEvent> disruptor) {
  disruptor.handleEventsWithWorkerPool(new C11EventHandler(), new C11EventHandler());
  disruptor.handleEventsWithWorkerPool(new C21EventHandler(), new C21EventHandler());
  disruptor.strat();
}
```

#### 6.1.6 航道模式

![disruptor19.png](https://agefades-note.oss-cn-beijing.aliyuncs.com/disruptor19.png)

```java
/**
 * 串行依次执行，同时c11、c21分别有两个实例
 */
public static void serialWithPool(Disruptor<LongEvent> disruptor) {
  disruptor.handleEventsWithWorkerPool(new C11EventHandler(), new C11EventHandler())
    .then(new C21EventHandler(), new C21EventHandler())
  disruptor.strat();
}
```

