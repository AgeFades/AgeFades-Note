[TOC]

# 杨过 - AQS同步器工具类

## Semaphore 

### 简介

- `Semaphore`：信号量
- `作用`：控制访问特定资源的线程数目
- 底层依赖 `AQS` 同步状态 `state` 
- 生产中比较常用的一个工具类

### 使用

#### 构造方法

```java
/**
 * permits 	许可线程的数量
 * fair			是否公平
 */
public Semaphore(int permits) {
  sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
  sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

#### 重要方法

```java
 /**
  * 阻塞并获取锁
  */
 public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
 }

 /**
  * 释放锁
  */
 public void release() {
        sync.releaseShared(1);
 }

/**
 * 阻塞尝试获取锁（在设置的超时时间内）
 */ 
public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

#### 使用场景

- 资源访问、服务限流
- 例如: `Hystrix` 限流就有基于 `信号量` 方式

#### 代码Demo

```java
import cn.hutool.core.date.DateUtil;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

/**
 * 信号量Demo
 *
 * @author DuChao
 * @date 2020/11/19 2:46 下午
 */
public class SemaphoreDemo {

    /**
     * 创建一次只允许两个线程持锁的信号量
     */
    private static final Semaphore semaphore = new Semaphore(2);

    public static void main(String[] args) {
        // 多线程模拟: 信号量对资源的限制
        System.out.println("信号量开始营业");
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println("当前时间: " + DateUtil.currentSeconds() + "\t" + Thread.currentThread().getName() + "持锁");
                    TimeUnit.SECONDS.sleep(1);
                    semaphore.release();
                    System.out.println("当前时间: " + DateUtil.currentSeconds() + "\t" + Thread.currentThread().getName() + "释放锁");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            },String.valueOf(i)).start();
        }
    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605769674422.png)

## CountDownLatch

### 简介

- `CountDownLatch` 能使 `一个线程` 等待 `其他线程完成工作后` 再执行
- 例如: 所有同学都离开教室了, 班长才能最后关门

### 使用场景

- Zookeeper分布式锁
- Jmeter模拟高并发
- ......

### 底层原理

- `CountDownLatch` 通过一个 `计数器` 来实现
- `计数器` 的初始值，为线程的数量
- 每当一个线程完成任务后，`计数器` 的值就会减1
- 当 `计数器` 的值为 `0` 时，表示所有线程已经完成任务，最后一个等待线程就可以恢复执行任务

### API

```java
/**
 * 线程完成任务，计数器减1
 */
public void countDown() {
    sync.releaseShared(1);
}

/**
 * 阻塞等待其他线程完成任务
 * 其他线程全部完成后，恢复执行
 */
public void await() throws InterruptedException {
  	sync.acquireSharedInterruptibly(1);
}
```

### 应用场景

- A任务: 做饭1秒
- B任务: 搞卫生1秒
- 打工人做 A、B 任务需要2秒
- 我就不同了，我有马仔，我只需要等这两人一块做完任务，带走就完事

### 代码Demo

```java
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.date.TimeInterval;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

/**
 * CountDownLatch Demo
 *
 * @author DuChao
 * @date 2020/11/19 3:23 下午
 */
public class CountDownLatchDemo {

    /**
     * 创建一个其他任务线程数为2的 countDownLatch
     */
    private static final CountDownLatch countDownLatch = new CountDownLatch(2);

    public static void main(String[] args) throws InterruptedException {

        TimeInterval timer = DateUtil.timer();

        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "做饭...");
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        },"马仔1号").start();

        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + "搞卫生...");
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                countDownLatch.countDown();
            }
        },"马仔2号").start();

        countDownLatch.await();
        System.out.println("打完收工，耗费时间: " + timer.intervalSecond() + "秒");
    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605771048758.png)

## CyclicBarrier

### 简介

- 让 `一组线程` 到达 `同步点` 时被阻塞
- 直到 `最后一个线程` 到达 `同步点` 时，所有阻塞线程才会被唤醒，继续执行任务

### 构造方法

```java
/**
 * parties  屏障拦截的线程数量
 */
public CyclicBarrier(int parties) {
  this(parties, null);
}
```

### API

```java
/**
 * 阻塞当前线程、等待所有线程到达 同步点
 */
public int await() throws InterruptedException, BrokenBarrierException {
    try {
      return dowait(false, 0L);
    } catch (TimeoutException toe) {
      throw new Error(toe); // cannot happen
    }
}
```

### 代码Demo

```java
import java.util.concurrent.CyclicBarrier;

/**
 * CyclicBarrier Demo
 *
 * @author DuChao
 * @date 2020/11/20 10:27 上午
 */
public class CyclicBarrierDemo {

    private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(7);

    public static void main(String[] args) throws Exception{
        for (int i = 1; i <= 7; i++) {
            new Thread(() -> {
                try {
                    System.out.println(Thread.currentThread().getName() + "到位!");
                    cyclicBarrier.await();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            },"葫芦娃" + i + "号").start();
        }

        cyclicBarrier.await();
        System.out.println("葫芦娃七兄弟已就绪，开始救爷爷!");
    }

}
```

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605839507581.png)

## Executors

### 简介

- 主要用来创建 `线程池`

### 重要方法

#### FixedThreadPool

```java
/**
 * nThreads 	核心线程数、最大线程数
 * 
 * 创建一个 线程数固定 的线程池
 * 可控制线程最大并发数，超出的线程会在队列中等待
 */
public static ExecutorService newFixedThreadPool(int nThreads) {
 	 return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```

#### SingleThreadExecutor

```java
/**
 * 创建一个 单线程工作 的线程池
 * 保证所有任务按照指定顺序（FIFO、LIFO）执行
 */
public static ExecutorService newSingleThreadExecutor() {
  return new FinalizableDelegatedExecutorService
          											(new ThreadPoolExecutor(1, 1,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()));
}
```

#### CachedThreadPool

```java
/**
 * 创建一个 线程数 灵活调整的线程池
 *
 * 任务紧张时、创建新线程处理任务
 * 线程闲置时、回收线程减少资源消耗
 */
public static ExecutorService newCachedThreadPool() {
  	return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                60L, TimeUnit.SECONDS,
                                new SynchronousQueue<Runnable>());
}
```

#### ScheduledThreadPool

```java
/**
 * 创建一个 线程数 固定的线程池
 *
 * 支持 定时 & 周期性任务 执行
 */ 
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

#### Exchanger

- 当一个线程运行到 exchange() 时会阻塞
- 另一个线程运行到 exchange() 时，二者交换数据，然后执行后面的程序
- 应用场景极少、知晓即可

