[TOC]

# 杨过 - BlockingQueue

## 概要

- `BlockingQueue` 是JUC包提供的用于解决 `并发 生产者-消费者` 问题的类，适用于许多场景
- 特性
  - 在任意时刻，只有一个线程可以进行 `take` 或者 `put` 操作
  - 提供超时 `return null` 机制

## 队列类型

- `无限队列` （unbounded queue）
  - 几乎可以无限增长，即: 可能导致 OOM 
- `有限队列` （bounded queue）
  - 定义了最大容量

## 队列数据结构

- `队列` ：一种存储数据的结构
  - 通常由 `链表` 或 `数组` 实现
  - 一般情况下，队列具备 `FIFO`（先进先出）的特性
  - 也有 `双端队列`（Deque）优先级队列
  - 主要操作: `入队`（EnQueue）与 `出队`（Dequeue）

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605680972802.png)

## 常见的4种阻塞队列

### ArrayBlockingQueue

#### 简介

- 由 `数组` 支持的 `有界队列`，容量大小在创建对象时就定义好了

#### 数据结构

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1605681096827.png)

#### 创建代码

```java
BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>();
```

#### 应用场景

- 在 `线程池` 中有较多应用，主要为 `生消模式`

#### 工作原理

- 基于 `ReentrantLock` 保证线程安全
- 根据 `Condition` 实现队列满时的 `阻塞`

#### 图

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608101156659.png)

#### 核心代码

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608100338779.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608100570056.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608100649871.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608100888937.png)

![](https://agefades-note.oss-cn-beijing.aliyuncs.com/1608106568445.png)

### LinkedBlockingQueue

#### 简介

- 基于 `链表` 的 `无界队列` （理论上有界）

#### 创建代码

```java
BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>();
```

#### 特性

- 上述代码中，blockingQueue 的容量将被设置为 `Integer.MAX_VALUE`
- 向 `无界队列` 添加元素的所有操作，将永远不会阻塞
  - `注意`：并不代表不会加锁来保证线程安全
- 使用 `无界队列` 设计 `生消模式` 时，最重要的是 `消费速率 >= 生产效率`
  - 否则，很可能导致 `OutOfMemoryException`

### DelayQueue

#### 简介

- 由 `优先级堆` 支持的、基于时间的 `调度队列`
- 内部基于无界队列 `PriorityQueue` 实现
- 而 `无界队列` 基于 `数组的扩容` 实现

#### 创建代码

```java
BlockingQueue<String> blockingQueue = new DelayQueue<>();
```

#### 要求

- 入队对象必须要实现 `Delayed` 接口，而 `Delayed` 继承自 `Comparable` 接口

#### 工作原理

- 队列内部会根据 `时间优先级` 进行排序
- 延迟类线程池周期执行

### PriorityBlockingQueue

#### 简介

- 由 `优先级堆` 支持的 `无界队列`

## BlockingQueue API

### 简介

- BlockingQueue 所有方法可以分为两大类
  - 添加元素
  - 检索元素
- 当队列 `满\空` 的情况下，来自这两组的每个方法行为都不同
- 这些API是 BlockingQueue 在构建 `生消模式` 最重要的函数

### 添加元素

| 方法                                    | 说明                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| add()                                   | 插入成功返回true，否则抛出 IllegalStateException             |
| put()                                   | 指定元素插入队列，如果队列满了，线程阻塞直至有空间可以插入   |
| offer()                                 | 插入成功返回true，否则返回false                              |
| offer(E e, long timeout, TimeUnit unit) | 插入成功返回true，否则阻塞(设置的超时时间)、期间有空间则插入成功，否则失败 |

### 检查元素

| 方法                              | 说明                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| take()                            | 获取队头并将其删除，如果队列为空，阻塞等待直至队列有元素     |
| poll(long timeout, TimeUnit unit) | 获取队头并将其删除，如果队列为空，阻塞等待(设置的超时时间)、期间队列有元素则返回，否则返回 null |

### 生消模式Demo

- 参考 Java 目录下 Java底层笔记

