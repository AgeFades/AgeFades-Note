# 杨过 - 抽象队列同步器AQS应用Lock详解

## 并发之父

- `Doug Lea` 在 JDK1.5 编写了整个 JUC并发工具包，JDK1.6之后被 JDK 纳入体系代码。
- 这也是Java 关键字 `Synchronized`自JDK1.6 优化升级的促因。

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605580571453.png)

## Java并发编程核心

- Java 并发编程核心在于 `java.concurrent.util`，简称 JUC
- JUC中，大多数同步器实现都是围绕着共同的基础行为，例如:
  - 等待队列
  - 条件队列
  - 独占获取
  - 共享获取
  - ......
- 而这些行为的抽象，就是基于 `AbstractQueuedSynchronizer`，简称 AQS
- AQS 定义了一套 `多线程访问共享资源` 的同步器框架，是一个 `依赖状态(state)` 的同步器

## ReentrantLock

- `ReentrantLock` 是一种基于 `AQS` 的应用实现，是JDK中的一种 `并发访问` 的同步手段
- 功能类似 `synchronized`，也是一种互斥锁，可以保证线程安全
- 具有比 `synchronized` 更丰富的特性，例如:
  - 支持 `手动加锁与解锁`
  - 支持 `加锁公平与非公平`
  - ......

### ReentrantLock同步Demo

```java
public class Demo {
  
  public static void mian(String[] args) {
     // 构造传参决定 是否公平锁。
     // false=非公平, true=公平
     ReentrantLock lock = new ReentrantLock(false);
    
     // 加锁
     lock.lock(); 
    
     // 解锁
     lock.unlock(); 
  }
  
}
```

### ReentrantLock如何实现公平与非公平？

- `ReentrantLock` 内部定义了一个 `Sync` 的内部类，该类继承 `AbstractQueuedSynchronizer`
- `Sync` 实现了该抽象类的部分方法，并且还定义了两个子类:
  - `FairSync` 公平锁的实现
  - `NonfairSync` 非公平锁的实现
- 主要使用的设计模式：`模板模式`，即 子类根据需要做具体业务实现

#### 源码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605581779006.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605581803615.png)

#### 差异

- `FairSync` 相比 `NonfairSync` 中 `tryAcquire` ，唯一多的就是一行判断 `!hasQueuedPredecessors()`

## AQS

### 特性

- 阻塞等待队列
- 共享/独占
- 公平/非公平
- 可重入
- 允许中断

### JUC对AQS的使用

- `JUC` 中同步器的实现，除了 `Lock` 外，诸如:
  - Latch
  - Barrier
  - BlockingQueue
  - ......
- 都是基于 `AQS` 框架实现
  - 一般通过定义内部类 `Sync` 继承 `AQS`
  - 将同步器所有调用都映射到 `Sync` 对应的方法
- `AQS` 内部维护属性 `volatile int state` (32位)
  - `state` 表示资源的可用状态
  - `state` 三种访问方式
    - `getState()`
    - `setState()`
    - `compareAndSetState()`
- `AQS` 定义两种 `资源共享` 方式
  - `Exclusive` 独占，只有一个线程能够执行，如 ReentrantLock
  - `Share` 共享，多个线程可以同时执行，如 Semaphore / CountDownLatch
- `AQS` 定义两种队列
  - `同步等待队列`
  - `条件等待队列`
- 不同的 `自定义同步器` 争用 `共享资源` 的方式也不同
- `自定义同步器` 在实现时，只需要实现 `共享资源state` 的获取和释放即可
- 至于具体线程 `等待队列` 的维护，如 `获取资源失败入队` / `唤醒出队` 等，`AQS` 已经在顶层实现好了
- `自定义同步器` 实现时，主要实现以下几种方法:
  - `isHeldExclusively()`：该线程是否正在独占资源，只有用到 `condition` 才需要去实现它
  - `tryAcquire(int)`：独占方式，尝试获取资源，成功返回true，失败返回false
  - `tryRelease(int)`：独占方式，尝试释放资源，成功返回true，失败返回false
  - `tryAcquireShared(int)`：共享方式，尝试获取资源。
    - 返回负数表示失败
    - 返回0表示成功，但没有剩余可用资源
    - 返回正数表示成功，且有剩余资源
  - `tryReleaseShared(int)`：共享方式，尝试释放资源。
    - 如果释放后，允许唤醒 `后续等待节点` 返回 true
    - 否则返回 false

### 同步等待队列

